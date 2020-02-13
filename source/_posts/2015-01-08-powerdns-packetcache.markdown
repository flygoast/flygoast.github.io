---
layout: post
title: "PowerDNS中PacketCache实现"
date: 2015-01-08 18:14:08 +0800
comments: true
categories: DNS
---
PowerDNS中DNS解析由各类Backend模块处理。如果解析相关的数据存储于MySQL, Postgres等数据库，Backend需要在这些数据库中查询相应的记录是否存在。查询数据库的性能很低。因而PowerDNS中实现了PacketCache来提高性能。PowerDNS接收到请求后，先在PacketCache中查询是否已经有相应的DNS响应。如果有则直接返回该缓存。否则交由backend处理，处理后再添加到PacketCache中。

PacketCache实现主要位于packetcache.hh和packetcache.cc中。

<!--more-->
common_startup.cc中定义了一些全局对象，其中包括一个PacketCache对象PC，PowerDNS中所有线程会共享这个对象。
```cpp
PacketCache PC; //!< This is the main PacketCache, shared accross all threads
```
来看PacketCache的构造函数，它初始化了一个读写锁和一些变量，接着添加了3个统计变量:
```cpp
PacketCache::PacketCache()
{
  pthread_rwlock_init(&d_mut, 0);
  // d_ops = 0;

  d_ttl=-1;
  d_recursivettl=-1;

  S.declare("packetcache-hit");
  S.declare("packetcache-miss");
  S.declare("packetcache-size");

  d_statnumhit=S.getPointer("packetcache-hit");
  d_statnummiss=S.getPointer("packetcache-miss");
  d_statnumentries=S.getPointer("packetcache-size");
}
```
PowerDNS有多个线程负责接收请求。线程的简化逻辑如下：
```cpp
  for(;;) {
    if(!(P=N->receive(&question))) { // receive a packet         inline
      continue;                    // packet was broken, try again
    }

    if(P->couldBeCached() && PC.get(P, &cached)) { // short circuit - does the PacketCache recognize this question?
      NS->send(&cached);   // answer it then
      continue;
    }

    distributor->question(P, &sendout);
  }
```
首先调用NameServer对象的receiver方法接收请求并解析到DNSPacket对象。接着调用DNSPacket::couldBeCached()判断请求是否可以被缓存, 可以看到PowerDNS只缓存Class为IN(Internet)的DNS请求。
```cpp
bool DNSPacket::couldBeCached()
{
  return d_ednsping.empty() && !d_wantsnsid && qclass==QClass::IN;
}
```
如果请求可以被缓存，则调用PC.get方法查询PacketCache中是否有相应缓存。
```cpp
int PacketCache::get(DNSPacket *p, DNSPacket *cached)
{
  extern StatBag S;

  if(d_ttl<0)
    getTTLS();

  if(!((++d_ops) % 300000)) {
    cleanup();
  }

  ...

  if(ntohs(p->d.qdcount)!=1) // we get confused by packets with more than one question
    return 0;

  string value;
  bool haveSomething;
  {
    TryReadLock l(&d_mut); // take a readlock here
    if(!l.gotIt()) {
      S.inc("deferred-cache-lookup");
      return 0;
    }

    uint16_t maxReplyLen = p->d_tcp ? 0xffff : p->getMaxReplyLen();
    haveSomething=getEntryLocked(p->qdomain, p->qtype, PacketCache::PACKETCACHE, value, -1, packetMeritsRecursion, maxReplyLen, p->d_dnssecOk, p->hasEDNS());
  }
  if(haveSomething) {
    (*d_statnumhit)++;
    if(cached->noparse(value.c_str(), value.size()) < 0) {
      return 0;
    }
    cached->spoofQuestion(p); // for correct case
    return 1;
  }

  //  cerr<<"Packet cache miss for '"<<p->qdomain<<"', merits: "<<packetMeritsRecursion<<endl;
  (*d_statnummiss)++;
  return 0; // bummer
}
```
get方法在第一次被调用时会调用getTTLS方法获取配置，"cache-ttl"为缓存有效时间。
```cpp
void PacketCache::getTTLS()
{
  d_ttl=::arg().asNum("cache-ttl");
  d_recursivettl=::arg().asNum("recursive-cache-ttl");

  d_doRecursion=::arg().mustDo("recursor");
}
```
每进行300000次查询操作(PacketCache::get和PacketCache::getEntry)时，get方法会调用一次cleanup函数。
```cpp
void PacketCache::cleanup()
{
  WriteLock l(&d_mut);

  *d_statnumentries=d_map.size();

  unsigned int maxCached=::arg().asNum("max-cache-entries");
  unsigned int toTrim=0;

  unsigned int cacheSize=*d_statnumentries;

  if(maxCached && cacheSize > maxCached) {
    toTrim = cacheSize - maxCached;
  }

  unsigned int lookAt=0;
  // two modes - if toTrim is 0, just look through 10%  of the cache and nuke everything that is expired
  // otherwise, scan first 5*toTrim records, and stop once we've nuked enough
  if(toTrim)
    lookAt=5*toTrim;
  else
    lookAt=cacheSize/10;

  //  cerr<<"cacheSize: "<<cacheSize<<", lookAt: "<<lookAt<<", toTrim: "<<toTrim<<endl;
  time_t now=time(0);

  DLOG(L<<"Starting cache clean"<<endl);
  if(d_map.empty())
    return; // clean

  typedef cmap_t::nth_index<1>::type sequence_t;
  sequence_t& sidx=d_map.get<1>();
  unsigned int erased=0, lookedAt=0;
  for(sequence_t::iterator i=sidx.begin(); i != sidx.end(); lookedAt++) {
    if(i->ttd < now) {
      sidx.erase(i++);
      erased++;
    }
    else
      ++i;

    if(toTrim && erased > toTrim)
      break;

    if(lookedAt > lookAt)
      break;
  }
  //  cerr<<"erased: "<<erased<<endl;
  *d_statnumentries=d_map.size();
  DLOG(L<<"Done with cache clean"<<endl);
}
```
cleanup函数首先获取"max-cache-entries"选项。如果配置了该选项，并且缓存数目已经超过该选项的值，则应从缓存的map结构中清除一部分过期缓存项。在清除时，搜寻个数为应清除个数的5倍。如果当前缓存个数没有超过“max-cache-entries", 则搜寻总个数的1/10。

个人感觉，这样实现并不太好，会导致这次请求处理时间过长。可以创建一个独立的线程周期性地清除缓存map中的过期项。

之后，get函数调用getEntryLocked来查找缓存中是否有该请求对应响应结果的缓存，通过map::find实现。
```cpp
bool PacketCache::getEntryLocked(const string &qname, const QType& qtype, CacheEntryType cet, string& value, int zoneID, bool meritsRecursion,
  unsigned int maxReplyLen, bool dnssecOK, bool hasEDNS)
{
  uint16_t qt = qtype.getCode();
  cmap_t::const_iterator i=d_map.find(tie(qname, qt, cet, zoneID, meritsRecursion, maxReplyLen, dnssecOK, hasEDNS));
  time_t now=time(0);
  bool ret=(i!=d_map.end() && i->ttd > now);
  if(ret)
    value = i->value;

  return ret;
}
```
如果从缓存中找到了响应数据包，则调用DNSPacket::noparse方法将结果保存到需要发回的响应包的相应结构中。
```cpp
int DNSPacket::noparse(const char *mesg, int length)
{
  d_rawpacket.assign(mesg,length);
  if(length < 12) {
    L << Logger::Warning << "Ignoring packet: too short from "
      << getRemote() << endl;
    return -1;
  }
  d_wantsnsid=false;
  d_ednsping.clear();
  d_maxreplylen=512;
  memcpy((void *)&d,(const void *)d_rawpacket.c_str(),12);
  return 0;
}
```
接着调用DNSPacket::spoofQuestion访求将请求包的请求域名部分复制到响应包的请求域名部分。
```cpp
void DNSPacket::spoofQuestion(const DNSPacket *qd)
{
  d_wrapped=true; // if we do this, don't later on wrapup

  int labellen;
  string::size_type i=sizeof(d);

  for(;;) {
    labellen = qd->d_rawpacket[i];
    if(!labellen) break;
    i++;
    d_rawpacket.replace(i, labellen, qd->d_rawpacket, i, labellen);
    i = i + labellen;
  }
}
```

个人感觉，请求域名部分的复制操作没有必要，缓存中的响应包的域名应该与本次请求包中的域名是相同的。

调用PC.get获取响应后，修改DNS头部的相应标志位及ID后，将响应发送出去。
```cpp
cached.d.rd=P->d.rd; // copy in recursion desired bit
cached.d.id=P->d.id;
cached.commitD();

N->send(&cached);
```
下面来看添加缓存的过程。

如果没有找到缓存，PowerDNS交由distributor处理DNS请求。
```cpp
distributor->question(P, &sendout); // otherwise, give to the distributor
```
这会调用PacketHandler::question函数。question函数会又会调用questionOrRecurse函数。questionOrRecurse函数在一系列检查后，对于CLASS为IN的请求，调用getAuth判断该DNS服务器是否是请求域名的授权服务器。
```cpp
bool PacketHandler::getAuth(DNSPacket *p, SOAData *sd, const string &target, int *zoneId)
{
  bool found=false;
  string subdomain(target);
  do {
    if( B.getSOA( subdomain, *sd, p ) ) {
      sd->qname = subdomain;
      if(zoneId)
        *zoneId = sd->domain_id;

      if(p->qtype.getCode() == QType::DS && pdns_iequals(subdomain, target)) {
        // Found authoritative zone but look for parent zone with 'DS' record.
        found=true;
      } else
        return true;
    }
  }
  while( chopOff( subdomain ) );   // 'www.powerdns.org' -> 'powerdns.org' -> 'org' -> ''
  return found;
}
```
getAuth函数对域名或其子域调用Backend的getSOA函数获得SOA记录，找到则返回。比如，请求解析`www.foo.com`，首先查找`www.foo.com`是否存在SOA记录。如果没有，接着查找"foo.com"是否存在SOA记录。直到查找部分为""。如果没有找到SOA记录，返回某种错误的响应。接下来，questionOrRecurse函数以`QTYPE::ANY`为参数调用Backend的lookup函数，并依次调用`B.get`获取找到的记录。
```cpp
B.lookup(QType(QType::ANY), target, p, sd.domain_id);
rrset.clear();
weDone = weRedirected = weHaveUnauth = 0;

while(B.get(rr)) {
  ...
  rrset.push_back(rr);
}
```
如果没有找到任何记录，则尝试泛解析，处理过程不详述。
```cpp
if(rrset.empty()) {
  if(tryWildcard(p, r, sd, target, wildcard, wereRetargeted, nodata)) {
    ...
  }
}
```
如果有CNAME记录，则重新针对CNAME目标域名进行上述逻辑。如果找到符合请求的记录，则添加到响应里发送出去。
```cpp
    if(weRedirected) {
      BOOST_FOREACH(rr, rrset) {
        if(rr.qtype.getCode() == QType::CNAME) {
          r->addRecord(rr);
          target = rr.content;
          retargetcount++;
          goto retargeted;
        }
      }
    }
    else if(weDone) {
      bool haveRecords = false;
      BOOST_FOREACH(rr, rrset) {
        if((p->qtype.getCode() == QType::ANY || rr.qtype == p->qtype) && rr.qtype.getCode() && rr.auth) {
          r->addRecord(rr);
          haveRecords = true;
        }
      }

      if (haveRecords) {
        if(p->qtype.getCode() == QType::ANY)
          completeANYRecords(p, r, sd, target);
      }
      else
        makeNOError(p, r, rr.qname, "", sd, 0);

      goto sendit;
    }
```
在发送前，会调用PC.insert将响应添加进PacketCache，实现通过map::insert。
```cpp
PC.insert(p, r, r->getMinTTL()); // in the packet cache
```
