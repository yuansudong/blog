---
title: 技术场景之优化邀请码
cover:  /img/technology-sense/optimize_live_room_title.png
subtitle: 技术场景之优化邀请码
categories: "技术场景"
tags: "技术场景"
author: 
  nick: 袁苏东
  link: https://github.com/yuansudong
---

互联网发展至今,人们对多媒体内容的需求从来都没有断过.直播就是其中的一种.随着直播的兴起,各种各样的直播间浮现在人们面前.比如计时付费直播间.



## 1.问题场景



直播间服务器,直白点就是房间. 一群具有相同特征人的聚集地.只负责直播间的业务.



而直播间的一个业务--计时扣费. 即过一段时间,对房间里的看客进行扣费.



最初,同事采用一个认证连接,一个协程,并且启动定时器的方式进行计时.通过定时器的触发管道.然后收到管道通知后,进行扣费.



```go
func (s *Session) startBilling(minutes int64) {
  duration := time.Minute * time.Duration(minutes)
  s.billingTimer = time.NewTimer(duration)
  defer s.billingTimer.Stop()
  for {
    select {
    case <-s.billingTimer.C:
      err := globals.userRepo.BillForTimeRoom(s.user, s.created, minutes, s.room.data)
      if err != nil {
        if err == perr.ErrSQLUpdateInvalid {
          s.notifyGoldNotEnough()
          s.ShutdownAll()
          return
        }
        log.Printf("billing fail for time room=%d, user=%d, minutes =%d", s.room.data.GetId(), s.user.GetId(), minutes)
      }
      s.queueOut(perr.ErrOutofGoldExplicitTs(s.user.GetId(), types.TimeStamp(), 0))
      s.billingTimer.Reset(duration)
    case <-s.billShutdown:
      return
    }
  }
}
```



首先,一个认证连接多一个协程. 协程在GO中就意味着内存. 



其次,一个认证连接多一个定时器. 当然, GO中的定时器.不是一个协程一个定时器.



而是,一个驱动器维持着一个四叉堆.当时间到达,通过一个缓冲容量为1的管道,通知SELECT该定时器管道的协程,让其执行任务.



如此做法,的确能实现直播间的计时扣费.但是,代价太大了.



## 解决办法



REDIS中有一个数据结构,名为有序集合(SORT SET). 经常被用作排行榜的场景.



但是,该数据结构还可以做一件事,那便是延时队列.或者说定时器.



即以过期时间戳作为分值,以认证连接的唯一ID作为成员,每次查询以[0,当前时间戳]进行查询,查回来的成员便是到达时间,需要处理扣费的连接.



如此做,将一个认证连接,一个协程,转移到了一个计时房间,一个计费协程.在一定程度上减少了协程,减少了内存,减少了GC的回收压力.



但是,此处还需要注意的是,对于轮询REDIS有序集合的频率.对于轮询频率的,我知道的有常数退避和指数退避



常数退避,即以固定时间休眠,休眠完成之后,再去对redis进行查询操作.



指数退避,即以1,2,4,4,4,4.... 的方式休眠,休眠之后再去对redis进行查询操作.



此处,我采用指数退避的方式减少轮询REDIS的频率.



下面是GOLANG的DEMO实现,达到抛砖引玉的目的.



```go
//  需要处理分页问题.
func (o *Object) _CoreBill() {
  o._WG.Add(1)
  mRds := globals.GetInstance().GetRedis()
  min, max, invalid := "0", "", 0
  sRdsKey := types.GetRedisBillKey(o._RoomData.RoomID)
  for {
    select {
    case <-o._BillShutdownChan:
      goto end
    default:
      max = fmt.Sprint(types.TimeStamp())
      suids, err := mRds.ZRangeByScore(
        context.Background(),
        sRdsKey,
        &redis.ZRangeBy{
          Min: min,
          Max: max,
        },
      ).Result()
      if err != nil {
        log.Println(err.Error())
      }
      iLength := len(suids)
      if iLength == 0 {
        iExp := invalid
        if invalid > 2 {
          iExp = 2
        }
        time.Sleep((1<<iExp + 1) * time.Second)
        invalid++
        continue
      }
      invalid = 0
      iExpire := time.Now().Add(time.Minute).Unix()
      for _, item := range suids {
        iIdentidy, _ := strconv.ParseInt(item, 10, 64)
        if iMem, ok := o._Set.Load(iIdentidy); ok {
          mMem := iMem.(*_Member)
          if mMem._Sess.GetUID() != o._RoomData.AnchorID {
            o._RoomData._BillForTimerRoom(mMem._Sess.GetUID(), 60)
            if _, err := mRds.ZAdd(
              context.TODO(),
              sRdsKey,
              &redis.Z{
                Score:  float64(iExpire),
                Member: mMem._Sess.GetIdentify(),
              },
            ).Result(); err != nil {
              log.Println(err.Error())
            }
          }
        }
      }
    }
  }
end:
  o._BillShutdownChan = nil
  o._WG.Done()
}
```



至此,解决完毕.