# 简介

## 情迁机器人
### 是什么
情迁机器人是一个仅在安卓运行的中转控制程序,简单来说它不是一个qq机器人,也不是一个微信机器人,或者电报机器人,你单纯下载本机器人并不能实现qq群管功能,而且下载本机器人也不会让你输入qq账户密码扫码登录等操作.
通过宿主和机器人构建协议,宿主app把收到的即时通讯传递给机器人,机器人执行一些逻辑之后把处理结果返回给宿主.


> 1.作者曾经通过Q++甚至内置QQ和本机器人软件进行绑定服务,从而实现了一套完整的QQ机器人.
> 
> 2.由于法律风险,Q++不再更新,不再对外发布,但是目前感兴趣的开发者可以加群体验一下机器人(https://jq.qq.com/?_wv=1027&k=KntMcRGt) 耐心阅读完教程,实现自己的一个"QQ机器人"
> 
> 3.不可利用本软件从事商业违法用途,仅供个人方便使用,由开发者实现了对应的控制能力并进行发布从事商业违法用途导致造成侵犯第三方企业经济损失的本人概不负责.
> 

### 原理
目前机器人实现了两套机制,
其原因正是因为宿主和情迁机器人是两个不同的应用,宿主和情迁机器人绑定的流程需要确保情迁机器人正常运行,因此存在诸多不稳定性问题,因此目前两套机制共同运行,确保提升体验.

1.宿主和情迁机器人绑定服务,拿到机器人内部```control api``` ,然后宿主添加自己的control回调,用于情迁机器人调用宿主,主要用于感知消息结果,批量处理人物消息,如需要禁言10个人,通过内容提供者就太过麻烦,对于处理结果的感知也不是很好.



如在```onChange```方法收到了某人发送的消息禁言 xxx,而机器人会自行调用control api的方法直接控制宿主,

2.直接调用内容提供者把收到的消息直接告知机器人,由机器人处理,然后通过安卓```ContentProvider```的```onChange```得到处理结果.


### 机器人能做什么?
本说明只从宏观角度来说,从开发者角度
不需要宿主在AndroidManifest.xml定义绑定服务等代码,因此非常适用于xposed插件场景
通过宿主执行搜图片的功能搜索图片
自带3套插件系统 支持java编写的插件,也支持lua ,js 编写的 插件
订阅股票行情
多个app监听消息或者xposed插件传递消息,传递指令给机器人然后机器人处理结果反馈给宿主
编写多个app的xposed插件传递消息个机器人然后机器人处理结果反馈给宿主
机器人内置楼层操控群能力,可快捷禁言

> 其他的情况自行开发xposed插件或者机器人插件均可实现对应的功能.
> 
> 作者目前实现了群订阅stock咨询消息,以及关键词自动禁言,搜索图片功能.批量禁言最近发言的人或者批量撤回最近发言的人或者批量撤回某个人所有消息.


# 宿主与机器人对接

## 监听机器人处理结果后的消息回调
在此逻辑主要是实现机器人的命令,比如机器人说我需要禁言,那么你判断类型是否是禁言是禁言你就控制你的即时通讯软件进行禁言即可.
下面是Q++的逻辑,各位自己实现大致也是可以这么写的.

创建内容提供者实现类
```

private static MyContentObserver mObserver = new ContentObserver(getHandler()) {
        @Override
        public boolean deliverSelfNotifications() {
            return super.deliverSelfNotifications();
        }
        @Override
        public void onChange(boolean selfChange) {
//                super.onChange(selfChange);
        }
        @Override
        public void onChange(boolean selfChange, Uri uri) {
//           Log.w("onChange, selfChange + "," + uri);

            if (selfChange) {
                return;
            }
            ;
            if (!QQConfigInner.enableRobotAssist) {
                Log.w(TAG, "沒有啟動機器人功能!");
                return;
            }
            if (BuildConfig.DEBUG) {
                Log.w(TAG, "onChange：" + uri);
            }
            MsgItem msgItem = RobotUtils.uriToMsgItem(uri);
              if (msgItem.getCode() == ControlCode.SUCC) {
                try {
                    replyContent(msgItem.getIstroop(), msgItem.getFrienduin(), msgItem.getSenderuin(), msgItem.getMessage());//群私聊的情况a=1000 另外b,c是群号 ，a是qqq 也就是frienduin而不是senderuin
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (InstantiationException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                } catch (InvocationTargetException e) {
                    e.printStackTrace();
                } catch (NoSuchMethodException e) {
                    e.printStackTrace();
                }
            } else if (msgItem.getCode() == ControlCode.GAG) {
                QQEngine.requestGag(msgItem.getFrienduin(), msgItem.getSenderuin(), Long.parseLong(msgItem.getMessage()));
            } else if (msgItem.getCode() == ControlCode.StrucMSG) {
                sendCardMsg(msgItem.getFrienduin(), msgItem.getSenderuin(), msgItem.getMessage(), msgItem.istroop);
            } else if (msgItem.getCode() == ControlCode.KICK) {
                requestKick(msgItem.getFrienduin(), msgItem.getSenderuin(), Boolean.parseBoolean(msgItem.getMessage()));
            } else if (msgItem.getCode() == ControlCode.SEND_PIC) {
                requestSendPic(msgItem);
            } else if (msgItem.getCode() == ControlCode.VOICE_CALL) {
                callVoice(QQEngine.getContext(), msgItem.getSenderuin(), msgItem.getNickname());
            } else if (msgItem.getCode() == ControlCode.ADD_LIKE) {
                requestAddLike(msgItem);
            } else if (msgItem.getCode() == ControlCode.QUIT_GROUP) {
                autoExitGroup(msgItem.getFrienduin());
            } else if (msgItem.getCode() == ControlCode.QUIT_DISCUSSION) {
                autoExitDiscussion(msgItem.getFrienduin());
            } else if (msgItem.getCode() == ControlCode.MODIFY_GROUP_MEMBER_CARD_NAME) {
                modifyGroupMemberCard(msgItem, msgItem.getMessage());
            } else if (msgItem.getCode() == ControlCode.REVOKE_MSG1) {
                doRevokeMsg(msgItem.getSenderuin(), msgItem.getFrienduin(), msgItem.getMessage(), msgItem.getMessageID());
            } else if (msgItem.getCode() == ControlCode.SEND_MIX_MSG) {
                ProtectedCode.sendMixedMsg(msgItem, msgItem.getMessage());
            } else if (msgItem.getCode() == ControlCode.TEST) {
            }
            }
        }
     
```

监听内容提供者

```
   mResolver = context.getContentResolver();
 try {
     Uri charNotifyUri = Uri.parse("content://cn.qssq666.robot");//注意,这个cn域名被弃用了,目前是lozn域名,将来我考虑使用top.lozn.robot...
     mResolver.registerContentObserver(charNotifyUri, true, mObserver);//mObserver是上面的实现类 MyContentObserver
 } catch (Throwable e) {
     Log.e(TAG, "注册机器人.失败", e);
 }

```
总结:宿主或者宿主xposed插件通过MsgHelper类发送消息给机器人,然后宿主又注册了```robot```的内容提供者```ContentResolver```因此能收到机器人的处理结果,根据处理结果然后宿主或者宿主xposed插件把机器人返回的消息进行处理,整个流程就完成了,
但是这样的流程会有一些弊端,比如对于批量禁言批量修改名片,通过此方法回调显然难以处理问题,但是此方法能解决大部分聊天逻辑了.


## 把收到的即时消息传递给机器人处理
> 此方法是走内容提供者通道
> 
> 此方法也适合非宿主的任意app执行此代码
> 
> 执行本代码只是把收到的指令传递给情迁机器人,然后情迁机器人再进行指令分发,比如机器人插件拦截了此消息,机器人插件进行处理,然后再转发交给宿主处理,
> 

作用:宿主收到了网友发送的聊天消息,这个消息需要传递给机器人,由机器人进行一些逻辑处理,机器人走完之后会通过上门说的```onChange```回调告诉你是需要禁言呢还是需要踢人还是需要让宿主发送消息(也就是机器人回复一条消息)呢?

***
案例

如果已经拥有Q++和情迁机器人绑定了服务的情况
我写了一个xposed插件,是专门监听某个app的通知然后转发给机器人的,把收到的通知发送给机器人处理,然后机器人再回调给q++,然后q++就执行了发送群消息的逻辑.
那么下面的代码就是可以做这么个事情,这个场景我目前一直在用,那就是把stock行情消息转发给qq群,实现其它并不难.


***

先拷贝下面代码, 分别创建MsgHelper 和 MsgTypeConstant类
```java
package cn.qssq666.main;

import android.content.ContentResolver;
import android.content.ContentValues;
import android.net.Uri;

import java.util.Date;

public class MsgHelper {
   public static Uri sendQQMsg(ContentResolver contentResolver, String robotQQ, String qq, String message){
      return   sendMsg(contentResolver,"插件",robotQQ,robotQQ,qq,message,0);
   }

   /**
    *
    * @param contentResolver
    * @param robotQQ
    * @param qqgroup
    * @param message
    * @return
    */
   public static Uri sendQQGroupMsg(ContentResolver contentResolver,String robotQQ,String friendUin,String qqgroup,String message){
      /**
       * 由于是机器人自己发送消息，所以senderuin 为 机器人自身。
       */
      return   sendMsg(contentResolver,"插件",robotQQ,friendUin,qqgroup,message,1);
   }

   /**
    * 仿造用户收到了发消息，让机器人处理。
    * @param contentResolver
    * @param nickname
    * @param selfAccount
    * @param account
    * @param frienduin
    * @param message
    * @param istroop
    * @return
    */
   public static Uri sendMsg(ContentResolver contentResolver, String nickname, String selfAccount, String account, String frienduin, String message, int istroop){
      Uri uri = Uri.withAppendedPath(Uri.parse(MsgTypeConstant.AUTHORITY_CONTENT), "insert/msg");
      ContentValues values = new ContentValues();
      values.put(MsgTypeConstant.msg, message);//消息内容
      values.put(MsgTypeConstant.nickname, nickname);//昵称
      values.put(MsgTypeConstant.time, new Date().getTime() / 1000);
      values.put(MsgTypeConstant.senderuin, account);//qq号码或者微信号码
      values.put(MsgTypeConstant.selfuin, selfAccount);//机器人自身的QQ号码，微信可以不填写
      values.put(MsgTypeConstant.frienduin, frienduin);//如果是群聊则是群号，否则填写QQ号码
      int MSG_TYPE_TEXT = -1000;//文本消息
      int type=MSG_TYPE_TEXT ;
      values.put(MsgTypeConstant.type, type);
      values.put(MsgTypeConstant.apptype, "test");
//        values.put(MsgTypeConstant.apptype, "proxy_send_msg");//这导致机器人会发重复的话。这是控制机器人发话的，因此不能用这个，
      values.put(MsgTypeConstant.time, new Date().getTime());
      values.put(MsgTypeConstant.istroop, istroop);//istroop =1代表群消息，否则代表私聊消息
      Uri insert = contentResolver.insert(uri, values);// 确保机器人已打开，正常情况下，回双向守护不会轻易宕机
      return insert;
   }

   /**
    * 让机器人发消息。
    * @param contentResolver
    * @param nickname
    * @param selfAccount
    * @param frienduin
    * @param message
    * @param istroop
    * @return
    */
   public static Uri robotSendMsg(ContentResolver contentResolver, String nickname, String selfAccount, String frienduin, String message, int istroop){
      Uri uri = Uri.withAppendedPath(Uri.parse(MsgTypeConstant.AUTHORITY_CONTENT), "insert/msg");
      ContentValues values = new ContentValues();
      values.put(MsgTypeConstant.msg, message);//消息内容
      values.put(MsgTypeConstant.nickname, nickname);//昵称
      values.put(MsgTypeConstant.time, new Date().getTime() / 1000);
      values.put(MsgTypeConstant.selfuin, selfAccount);//让机器人自己发送消息，所以这里就是自己，全部填写自己
      values.put(MsgTypeConstant.senderuin,selfAccount);//qq号码或者微信号码
      values.put(MsgTypeConstant.frienduin, frienduin);//如果是群聊则是群号，否则填写QQ号码
      int MSG_TYPE_TEXT = -1000;//文本消息
      int type=MSG_TYPE_TEXT ;
      values.put(MsgTypeConstant.type, type);
      values.put(MsgTypeConstant.apptype, "proxy_send_msg");//这是控制机器人发话的，
      values.put(MsgTypeConstant.time, new Date().getTime());
      values.put(MsgTypeConstant.istroop, istroop);//istroop =1代表群消息，否则代表私聊消息
      Uri insert = contentResolver.insert(uri, values);// 确保机器人已打开，正常情况下，回双向守护不会轻易宕机
      return insert;
   }
}


package cn.qssq666.main;

public interface MsgTypeConstant {


   String nickname = "nickname";
   String extstr = "extstr";

   public final static String ACTION_MSG = "insert/msg";//表示这目录下面所有
   String AUTHORITY = "cn.qssq666.robot";
   String AUTHORITY_CONTENT = "content://" + AUTHORITY;
   public final static String ACTION_GAD = "insert/gad";//表示这目录下面所有
   public final static String ACTION_KICK = "insert/kick";
   public final static String ERROR_JSON = "{'msg':'error',code:-1}";
   String istroop = "istroop";
   String version = "version";
   String time = "time";
   String senderuin = "senderuin";
   String frienduin = "frienduin";
   String selfuin = "selfuin";
   String type = "type";
   String code = "code";
   String apptype = "apptype";
   String extrajson = "extrajson";
   String msg = "msg";
   int MSG_TYPE_TEXT = -1000;
   int MSG_TYPE_SHUOSHUO = -2015;
   int MSG_TYPE_PIC = -2000;
   int MSG_TYPE_PIC_WITH_TEXT = -1035;
   int MSG_TYPE_REDPACKET = -2025;
   int MSG_TYPE_REDPACKET_1 = -2500;
}



```
下面是执行发送的代码
```java

 Uri uri = Uri.withAppendedPath(Uri.parse(MsgTypeConstant.AUTHORITY_CONTENT), “insert/msg”);
        Log.w(TAG, "sendMsg:" + uri.toString());
        ContentValues values = new ContentValues();
        values.put(MsgTypeConstant.msg, message);//消息内容
        values.put(MsgTypeConstant.nickname, nickname);//昵称
        values.put(MsgTypeConstant.time, new Date().getTime() / 1000);
        values.put(MsgTypeConstant.senderuin, senderuin);//qq号码或者微信号码
        values.put(MsgTypeConstant.selfuin, selfuin);//机器人自身的QQ号码，微信可以不填写
        values.put(MsgTypeConstant.frienduin, frienduin);//如果是群聊则是群号，否则填写QQ号码
        int MSG_TYPE_TEXT = -99999;//文本消息
        int type=MSG_TYPE_TEXT ;
        values.put(MsgTypeConstant.type, type);
        values.put(MsgTypeConstant.apptype, "test");
        values.put(MsgTypeConstant.time, new Date().getTime());
        values.put(MsgTypeConstant.istroop, istroop);//istroop =1代表群消息，否则代表私聊消息
        Uri insert = resolver.insert(uri, values);// 确保机器人已打开，正常情况下，回双向守护不会轻易宕机
        
```
实际场景是我在stock的插件里面写了这样的逻辑
hook通知栏,然后把通知栏的内容取出,进行发送
``` 

                  Notification notification = (Notification) param.args[2];
                    String aPackage = notification.contentView.getPackage();
                    String title = "";
                    String text = "";
                    Bundle bundle = null;
                    if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.KITKAT) {
                        bundle = notification.extras;
                        title = (String) bundle.get("android.title");
                        text = (String) bundle.get("android.text");
                    }
                  StringBuffer sb=new StringBuffer();
                  sb.append("\n标题:" + title);
                  sb.append("\内容:" + text);
                  String s=sb.toString();
                  Log.w(TAG, "notifyAsUser" + s.toString() + Log.getStackTraceString(new Exception()));
                    if (!s.contains("HOOK")) {//hookui的通知忽略
                        NotificationManager notificationManager = (NotificationManager) param.thisObject;
                        MsgHelper.robotSendMsg(cn.qssq666.hotupdate.XposeUtil.getApplication().getContentResolver(), "notification_bar", Main.selfqq, Main.group, s, 1);
                        //1 是群消息 group是群号 selfqq是 机器人qq账户, s是消息内容  notification_bar 是昵称.
                    }
```





## 高级-情迁机器人反向控制宿主执行批量操作
有这个一个场景,我需要批量修改所有的好友备注,我首先需要调用宿主的api得到所有好友.然后再调用宿主api的修改名片接口批量修改,并且把操作结果是否成功失败告知,这种逻辑上面的场景是无法实现的,下面来看看机器人绑定服务的方式怎么实现吧.
### 机器人的进程通讯文件概览
(可忽略不看)因为集成不需要修改机器人任何代码
机器人的aidl文件
ICallBack.aidl文件
``` 

package cn.qssq666.robot;
import java.util.List;
import java.util.Map;
interface ICallBack {
    void actionPerformed (int actionId);
//基本数据类型默认为in  can be an out type, so you must declare it as in, out or inout. 'out String flag2' can only be an in parameter.
     void onReceiveMsg( int flag, int what,boolean flag1,   String flag2,in Map map,in List list);
     Map queryClientData(int flag,int waht,boolean flag1,String flag2,in Map map,in List list);
     String queryData(int flag,int what,in String[] arg);
     String queryDataArr(int flag,int what,in String[] arg,in int[] intarg);
  String queryDataByMap(int flag,int what,in String[] arg,in int[] intarg,in Map map);
     Map queryData2Map(int flag,int what,in String[] arg,in int[] intarg);
     List queryData2List(int flag,int what,in String[] arg,in int[] intarg,in boolean[] boolflag);
     boolean execBooleanAction(int flag,int what,in String[] arg,in Map arg1);

}
```

RobotCallBinder.aidl文件
``` 

// RobotCall.aidl
package cn.qssq666.robot;
import cn.qssq666.robot.ICallback;

// Declare any non-default types here with import statements

interface RobotCallBinder {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
      void clearCallBack( );
    void registerCallback(in ICallBack cb);
    void unregisterCallback( in ICallBack cb);
     boolean isTaskRunning();
      void stopRunningTask();
        String queryConfig1(int flag,int what,in String[] arg);
        String queryConfig2(int flag,int what,in String[] arg);
        String queryConfig3(int flag,int what,in String[] arg);
        boolean queryEnable1(int flag,int what,in String[] arg);
        boolean queryEnable2(int flag,int what,in String[] arg);
        boolean queryEnable3(int flag,int what,in String[] arg);
        List queryData(int action,boolean flag1,String flag2);
        List queryDataStr(int action,boolean flag1,in String[] flag);
        Map queryMapData(int action,boolean flag1,String flag2);
        String queryDataByMap(int flag,int what,in String[] arg,in int[] intarg,in Map map);
        Map queryData2Map(int flag,int what,in String[] arg,in int[] intarg);
        List queryData2List(int flag,int what,in String[] arg,in int[] intarg,in boolean[] boolflag);

        int getPid();
        int getVersionCode();
        String getVersionName();
        String getLoginUser();
        String getLoginToken();
        boolean isAuthorUser();
        boolean isLogin();
        String getMenuStr();
        String getHostInfo();

}

```


GeneralBinder文件
``` 
 // RobotCall.aidl
package cn.qssq666;
import cn.qssq666.robot.ICallback;
interface GeneralBinder {
    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);
      void clearCallBack( );
     boolean isTaskRunning();
      void stopRunningTask();
        String queryConfig1(int flag,int what,in String[] arg);
        String queryConfig2(int flag,int what,in String[] arg);
        String queryConfig3(int flag,int what,in String[] arg);
        boolean queryEnable1(int flag,int what,in String[] arg);
        boolean queryEnable2(int flag,int what,in String[] arg);
        boolean queryEnable3(int flag,int what,in String[] arg);
        List queryData(int action,boolean flag1,String flag2);
        List queryDataStr(int action,boolean flag1,in String[] flag);
        Map queryMapData(int action,boolean flag1,String flag2);
        String queryDataByMap(int flag,int what,in String[] arg,in int[] intarg,in Map map);
        Map queryData2Map(int flag,int what,in String[] arg,in int[] intarg);
        List queryData2List(int flag,int what,in String[] arg,in int[] intarg,in boolean[] boolflag);
        int getInt();
        int getIntVargs(int arg,int arg2,int arg3,int arg4);
        String getStr();
        String getStrVars(String arg,String arg1,String arg2,String Arg3);
        void setIntVargs(int arg,int arg2,int arg3,int arg4);
        String setStrVars(String arg,String arg1,String arg2,String Arg3);

}
 
 ```

### 定义RobotCallBinder

``` 

package cn.qssq666.robot;
// Declare any non-default types here with import statements

public interface RobotCallBinder extends android.os.IInterface
{
    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements cn.qssq666.robot.RobotCallBinder
    {
        private static final java.lang.String DESCRIPTOR = "cn.qssq666.robot.RobotCallBinder";
        /** Construct the stub at attach it to the interface. */
        public Stub()
        {
            this.attachInterface(this, DESCRIPTOR);
        }
        /**
         * Cast an IBinder object into an cn.qssq666.robot.RobotCallBinder interface,
         * generating a proxy if needed.
         */
        public static cn.qssq666.robot.RobotCallBinder asInterface(android.os.IBinder obj)
        {
            if ((obj==null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin!=null)&&(iin instanceof cn.qssq666.robot.RobotCallBinder))) {
                return ((cn.qssq666.robot.RobotCallBinder)iin);
            }
            return new cn.qssq666.robot.RobotCallBinder.Stub.Proxy(obj);
        }
        @Override public android.os.IBinder asBinder()
        {
            return this;
        }
        @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
        {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code)
            {
                case INTERFACE_TRANSACTION:
                {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_basicTypes:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    long _arg1;
                    _arg1 = data.readLong();
                    boolean _arg2;
                    _arg2 = (0!=data.readInt());
                    float _arg3;
                    _arg3 = data.readFloat();
                    double _arg4;
                    _arg4 = data.readDouble();
                    java.lang.String _arg5;
                    _arg5 = data.readString();
                    this.basicTypes(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_clearCallBack:
                {
                    data.enforceInterface(descriptor);
                    this.clearCallBack();
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_registerCallback:
                {
                    data.enforceInterface(descriptor);
                    cn.qssq666.robot.ICallBack _arg0;
                    _arg0 = cn.qssq666.robot.ICallBack.Stub.asInterface(data.readStrongBinder());
                    this.registerCallback(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_unregisterCallback:
                {
                    data.enforceInterface(descriptor);
                    cn.qssq666.robot.ICallBack _arg0;
                    _arg0 = cn.qssq666.robot.ICallBack.Stub.asInterface(data.readStrongBinder());
                    this.unregisterCallback(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_isTaskRunning:
                {
                    data.enforceInterface(descriptor);
                    boolean _result = this.isTaskRunning();
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_stopRunningTask:
                {
                    data.enforceInterface(descriptor);
                    this.stopRunningTask();
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_queryConfig1:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    java.lang.String _result = this.queryConfig1(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_queryConfig2:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    java.lang.String _result = this.queryConfig2(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_queryConfig3:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    java.lang.String _result = this.queryConfig3(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_queryEnable1:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    boolean _result = this.queryEnable1(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_queryEnable2:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    boolean _result = this.queryEnable2(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_queryEnable3:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    boolean _result = this.queryEnable3(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_queryData:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    boolean _arg1;
                    _arg1 = (0!=data.readInt());
                    java.lang.String _arg2;
                    _arg2 = data.readString();
                    java.util.List _result = this.queryData(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeList(_result);
                    return true;
                }
                case TRANSACTION_queryDataStr:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    boolean _arg1;
                    _arg1 = (0!=data.readInt());
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    java.util.List _result = this.queryDataStr(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeList(_result);
                    return true;
                }
                case TRANSACTION_queryMapData:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    boolean _arg1;
                    _arg1 = (0!=data.readInt());
                    java.lang.String _arg2;
                    _arg2 = data.readString();
                    java.util.Map _result = this.queryMapData(_arg0, _arg1, _arg2);
                    reply.writeNoException();
                    reply.writeMap(_result);
                    return true;
                }
                case TRANSACTION_queryDataByMap:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    int[] _arg3;
                    _arg3 = data.createIntArray();
                    java.util.Map _arg4;
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _arg4 = data.readHashMap(cl);
                    java.lang.String _result = this.queryDataByMap(_arg0, _arg1, _arg2, _arg3, _arg4);
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_queryData2Map:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    int[] _arg3;
                    _arg3 = data.createIntArray();
                    java.util.Map _result = this.queryData2Map(_arg0, _arg1, _arg2, _arg3);
                    reply.writeNoException();
                    reply.writeMap(_result);
                    return true;
                }
                case TRANSACTION_queryData2List:
                {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    int _arg1;
                    _arg1 = data.readInt();
                    java.lang.String[] _arg2;
                    _arg2 = data.createStringArray();
                    int[] _arg3;
                    _arg3 = data.createIntArray();
                    boolean[] _arg4;
                    _arg4 = data.createBooleanArray();
                    java.util.List _result = this.queryData2List(_arg0, _arg1, _arg2, _arg3, _arg4);
                    reply.writeNoException();
                    reply.writeList(_result);
                    return true;
                }
                case TRANSACTION_getPid:
                {
                    data.enforceInterface(descriptor);
                    int _result = this.getPid();
                    reply.writeNoException();
                    reply.writeInt(_result);
                    return true;
                }
                case TRANSACTION_getVersionCode:
                {
                    data.enforceInterface(descriptor);
                    int _result = this.getVersionCode();
                    reply.writeNoException();
                    reply.writeInt(_result);
                    return true;
                }
                case TRANSACTION_getVersionName:
                {
                    data.enforceInterface(descriptor);
                    java.lang.String _result = this.getVersionName();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_getLoginUser:
                {
                    data.enforceInterface(descriptor);
                    java.lang.String _result = this.getLoginUser();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_getLoginToken:
                {
                    data.enforceInterface(descriptor);
                    java.lang.String _result = this.getLoginToken();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_isAuthorUser:
                {
                    data.enforceInterface(descriptor);
                    boolean _result = this.isAuthorUser();
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_isLogin:
                {
                    data.enforceInterface(descriptor);
                    boolean _result = this.isLogin();
                    reply.writeNoException();
                    reply.writeInt(((_result)?(1):(0)));
                    return true;
                }
                case TRANSACTION_getMenuStr:
                {
                    data.enforceInterface(descriptor);
                    java.lang.String _result = this.getMenuStr();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                case TRANSACTION_getHostInfo:
                {
                    data.enforceInterface(descriptor);
                    java.lang.String _result = this.getHostInfo();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
                default:
                {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
        private static class Proxy implements cn.qssq666.robot.RobotCallBinder
        {
            private android.os.IBinder mRemote;
            Proxy(android.os.IBinder remote)
            {
                mRemote = remote;
            }
            @Override public android.os.IBinder asBinder()
            {
                return mRemote;
            }
            public java.lang.String getInterfaceDescriptor()
            {
                return DESCRIPTOR;
            }
            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(anInt);
                    _data.writeLong(aLong);
                    _data.writeInt(((aBoolean)?(1):(0)));
                    _data.writeFloat(aFloat);
                    _data.writeDouble(aDouble);
                    _data.writeString(aString);
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public void clearCallBack() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_clearCallBack, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public void registerCallback(cn.qssq666.robot.ICallBack cb) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeStrongBinder((((cb!=null))?(cb.asBinder()):(null)));
                    mRemote.transact(Stub.TRANSACTION_registerCallback, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public void unregisterCallback(cn.qssq666.robot.ICallBack cb) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeStrongBinder((((cb!=null))?(cb.asBinder()):(null)));
                    mRemote.transact(Stub.TRANSACTION_unregisterCallback, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public boolean isTaskRunning() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_isTaskRunning, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public void stopRunningTask() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_stopRunningTask, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public java.lang.String queryConfig1(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryConfig1, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String queryConfig2(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryConfig2, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String queryConfig3(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryConfig3, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public boolean queryEnable1(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryEnable1, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public boolean queryEnable2(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryEnable2, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public boolean queryEnable3(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    mRemote.transact(Stub.TRANSACTION_queryEnable3, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.util.List queryData(int action, boolean flag1, java.lang.String flag2) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(action);
                    _data.writeInt(((flag1)?(1):(0)));
                    _data.writeString(flag2);
                    mRemote.transact(Stub.TRANSACTION_queryData, _data, _reply, 0);
                    _reply.readException();
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _result = _reply.readArrayList(cl);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.util.List queryDataStr(int action, boolean flag1, java.lang.String[] flag) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(action);
                    _data.writeInt(((flag1)?(1):(0)));
                    _data.writeStringArray(flag);
                    mRemote.transact(Stub.TRANSACTION_queryDataStr, _data, _reply, 0);
                    _reply.readException();
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _result = _reply.readArrayList(cl);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.util.Map queryMapData(int action, boolean flag1, java.lang.String flag2) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.Map _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(action);
                    _data.writeInt(((flag1)?(1):(0)));
                    _data.writeString(flag2);
                    mRemote.transact(Stub.TRANSACTION_queryMapData, _data, _reply, 0);
                    _reply.readException();
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _result = _reply.readHashMap(cl);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String queryDataByMap(int flag, int what, java.lang.String[] arg, int[] intarg, java.util.Map map) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    _data.writeIntArray(intarg);
                    _data.writeMap(map);
                    mRemote.transact(Stub.TRANSACTION_queryDataByMap, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.util.Map queryData2Map(int flag, int what, java.lang.String[] arg, int[] intarg) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.Map _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    _data.writeIntArray(intarg);
                    mRemote.transact(Stub.TRANSACTION_queryData2Map, _data, _reply, 0);
                    _reply.readException();
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _result = _reply.readHashMap(cl);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.util.List queryData2List(int flag, int what, java.lang.String[] arg, int[] intarg, boolean[] boolflag) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(flag);
                    _data.writeInt(what);
                    _data.writeStringArray(arg);
                    _data.writeIntArray(intarg);
                    _data.writeBooleanArray(boolflag);
                    mRemote.transact(Stub.TRANSACTION_queryData2List, _data, _reply, 0);
                    _reply.readException();
                    java.lang.ClassLoader cl = (java.lang.ClassLoader)this.getClass().getClassLoader();
                    _result = _reply.readArrayList(cl);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public int getPid() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getPid, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public int getVersionCode() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getVersionCode, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String getVersionName() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getVersionName, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String getLoginUser() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getLoginUser, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String getLoginToken() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getLoginToken, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public boolean isAuthorUser() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_isAuthorUser, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public boolean isLogin() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_isLogin, _data, _reply, 0);
                    _reply.readException();
                    _result = (0!=_reply.readInt());
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String getMenuStr() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getMenuStr, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
            @Override public java.lang.String getHostInfo() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getHostInfo, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
        static final int TRANSACTION_basicTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_clearCallBack = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_registerCallback = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        static final int TRANSACTION_unregisterCallback = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
        static final int TRANSACTION_isTaskRunning = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
        static final int TRANSACTION_stopRunningTask = (android.os.IBinder.FIRST_CALL_TRANSACTION + 5);
        static final int TRANSACTION_queryConfig1 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 6);
        static final int TRANSACTION_queryConfig2 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 7);
        static final int TRANSACTION_queryConfig3 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 8);
        static final int TRANSACTION_queryEnable1 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 9);
        static final int TRANSACTION_queryEnable2 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 10);
        static final int TRANSACTION_queryEnable3 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 11);
        static final int TRANSACTION_queryData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 12);
        static final int TRANSACTION_queryDataStr = (android.os.IBinder.FIRST_CALL_TRANSACTION + 13);
        static final int TRANSACTION_queryMapData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 14);
        static final int TRANSACTION_queryDataByMap = (android.os.IBinder.FIRST_CALL_TRANSACTION + 15);
        static final int TRANSACTION_queryData2Map = (android.os.IBinder.FIRST_CALL_TRANSACTION + 16);
        static final int TRANSACTION_queryData2List = (android.os.IBinder.FIRST_CALL_TRANSACTION + 17);
        static final int TRANSACTION_getPid = (android.os.IBinder.FIRST_CALL_TRANSACTION + 18);
        static final int TRANSACTION_getVersionCode = (android.os.IBinder.FIRST_CALL_TRANSACTION + 19);
        static final int TRANSACTION_getVersionName = (android.os.IBinder.FIRST_CALL_TRANSACTION + 20);
        static final int TRANSACTION_getLoginUser = (android.os.IBinder.FIRST_CALL_TRANSACTION + 21);
        static final int TRANSACTION_getLoginToken = (android.os.IBinder.FIRST_CALL_TRANSACTION + 22);
        static final int TRANSACTION_isAuthorUser = (android.os.IBinder.FIRST_CALL_TRANSACTION + 23);
        static final int TRANSACTION_isLogin = (android.os.IBinder.FIRST_CALL_TRANSACTION + 24);
        static final int TRANSACTION_getMenuStr = (android.os.IBinder.FIRST_CALL_TRANSACTION + 25);
        static final int TRANSACTION_getHostInfo = (android.os.IBinder.FIRST_CALL_TRANSACTION + 26);
    }
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat, double aDouble, java.lang.String aString) throws android.os.RemoteException;
    public void clearCallBack() throws android.os.RemoteException;
    public void registerCallback(cn.qssq666.robot.ICallBack cb) throws android.os.RemoteException;
    public void unregisterCallback(cn.qssq666.robot.ICallBack cb) throws android.os.RemoteException;
    public boolean isTaskRunning() throws android.os.RemoteException;
    public void stopRunningTask() throws android.os.RemoteException;
    public java.lang.String queryConfig1(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public java.lang.String queryConfig2(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public java.lang.String queryConfig3(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public boolean queryEnable1(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public boolean queryEnable2(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public boolean queryEnable3(int flag, int what, java.lang.String[] arg) throws android.os.RemoteException;
    public java.util.List queryData(int action, boolean flag1, java.lang.String flag2) throws android.os.RemoteException;
    public java.util.List queryDataStr(int action, boolean flag1, java.lang.String[] flag) throws android.os.RemoteException;
    public java.util.Map queryMapData(int action, boolean flag1, java.lang.String flag2) throws android.os.RemoteException;
    public java.lang.String queryDataByMap(int flag, int what, java.lang.String[] arg, int[] intarg, java.util.Map map) throws android.os.RemoteException;
    public java.util.Map queryData2Map(int flag, int what, java.lang.String[] arg, int[] intarg) throws android.os.RemoteException;
    public java.util.List queryData2List(int flag, int what, java.lang.String[] arg, int[] intarg, boolean[] boolflag) throws android.os.RemoteException;
    public int getPid() throws android.os.RemoteException;
    public int getVersionCode() throws android.os.RemoteException;
    public java.lang.String getVersionName() throws android.os.RemoteException;
    public java.lang.String getLoginUser() throws android.os.RemoteException;
    public java.lang.String getLoginToken() throws android.os.RemoteException;
    public boolean isAuthorUser() throws android.os.RemoteException;
    public boolean isLogin() throws android.os.RemoteException;
    public java.lang.String getMenuStr() throws android.os.RemoteException;
    public java.lang.String getHostInfo() throws android.os.RemoteException;
}

```
### 定义ICallback类
在你的宿主app中定义如下类ICallBack
```  
ackage cn.qssq666.robot;
import cn.qssq666.EncryptUtilN; 
public interface ICallBack extends android.os.IInterface
{
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements cn.qssq666.robot.ICallBack
{
private static final String DESCRIPTOR = "cn.qssq666.robot.ICallBack";
/** Construct the stub at attach it to the interface. */
public Stub()
{
this.attachInterface(this, DESCRIPTOR);
}
/**
 * Cast an IBinder object into an cn.qssq666.robot.ICallBack interface,
 * generating a proxy if needed.
 */
public static cn.qssq666.robot.ICallBack asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof cn.qssq666.robot.ICallBack))) {
return ((cn.qssq666.robot.ICallBack)iin);
}
return new cn.qssq666.robot.ICallBack.Stub.Proxy(obj);
}
@Override public android.os.IBinder asBinder()
{
return this;
}
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)
{
case INTERFACE_TRANSACTION:
{
reply.writeString(DESCRIPTOR);
return true;
}
case TRANSACTION_actionPerformed:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
this.actionPerformed(_arg0);
reply.writeNoException();
return true;
}
case TRANSACTION_onReceiveMsg:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
boolean _arg2;
_arg2 = (0!=data.readInt());
String _arg3;
_arg3 = data.readString();
java.util.Map _arg4;
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_arg4 = data.readHashMap(cl);
java.util.List _arg5;
_arg5 = data.readArrayList(cl);
this.onReceiveMsg(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
reply.writeNoException();
return true;
}
case TRANSACTION_queryClientData:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
boolean _arg2;
_arg2 = (0!=data.readInt());
String _arg3;
_arg3 = data.readString();
java.util.Map _arg4;
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_arg4 = data.readHashMap(cl);
java.util.List _arg5;
_arg5 = data.readArrayList(cl);
java.util.Map _result = this.queryClientData(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5);
reply.writeNoException();
reply.writeMap(_result);
return true;
}
case TRANSACTION_queryData:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
String _result = this.queryData(_arg0, _arg1, _arg2);
reply.writeNoException();
reply.writeString(_result);
return true;
}
case TRANSACTION_queryDataArr:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
int[] _arg3;
_arg3 = data.createIntArray();
String _result = this.queryDataArr(_arg0, _arg1, _arg2, _arg3);
reply.writeNoException();
reply.writeString(_result);
return true;
}
case TRANSACTION_queryDataByMap:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
int[] _arg3;
_arg3 = data.createIntArray();
java.util.Map _arg4;
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_arg4 = data.readHashMap(cl);
String _result = this.queryDataByMap(_arg0, _arg1, _arg2, _arg3, _arg4);
reply.writeNoException();
reply.writeString(_result);
return true;
}
case TRANSACTION_queryData2Map:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
int[] _arg3;
_arg3 = data.createIntArray();
java.util.Map _result = this.queryData2Map(_arg0, _arg1, _arg2, _arg3);
reply.writeNoException();
reply.writeMap(_result);
return true;
}
case TRANSACTION_queryData2List:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
int[] _arg3;
_arg3 = data.createIntArray();
boolean[] _arg4;
_arg4 = data.createBooleanArray();
java.util.List _result = this.queryData2List(_arg0, _arg1, _arg2, _arg3, _arg4);
reply.writeNoException();
reply.writeList(_result);
return true;
}
case TRANSACTION_execBooleanAction:
{
data.enforceInterface(DESCRIPTOR);
int _arg0;
_arg0 = data.readInt();
int _arg1;
_arg1 = data.readInt();
String[] _arg2;
_arg2 = data.createStringArray();
java.util.Map _arg3;
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_arg3 = data.readHashMap(cl);
boolean _result = this.execBooleanAction(_arg0, _arg1, _arg2, _arg3);
reply.writeNoException();
reply.writeInt(((_result)?(1):(0)));
return true;
}
}
return super.onTransact(code, data, reply, flags);
}
private static class Proxy implements cn.qssq666.robot.ICallBack
{
private android.os.IBinder mRemote;
Proxy(android.os.IBinder remote)
{
mRemote = remote;
}
@Override public android.os.IBinder asBinder()
{
return mRemote;
}
public String getInterfaceDescriptor()
{
return DESCRIPTOR;
}
@Override public void actionPerformed(int actionId) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(actionId);
mRemote.transact(Stub.TRANSACTION_actionPerformed, _data, _reply, 0);
_reply.readException();
}
finally {
_reply.recycle();
_data.recycle();
}
}
//基本数据类型默认为in  can be an out type, so you must declare it as in, out or inout. 'out String flag2' can only be an in parameter.

@Override public void onReceiveMsg(int flag, int what, boolean flag1, String flag2, java.util.Map map, java.util.List list) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeInt(((flag1)?(1):(0)));
_data.writeString(flag2);
_data.writeMap(map);
_data.writeList(list);
mRemote.transact(Stub.TRANSACTION_onReceiveMsg, _data, _reply, 0);
_reply.readException();
}
finally {
_reply.recycle();
_data.recycle();
}
}
@Override public java.util.Map queryClientData(int flag, int waht, boolean flag1, String flag2, java.util.Map map, java.util.List list) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
java.util.Map _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(waht);
_data.writeInt(((flag1)?(1):(0)));
_data.writeString(flag2);
_data.writeMap(map);
_data.writeList(list);
mRemote.transact(Stub.TRANSACTION_queryClientData, _data, _reply, 0);
_reply.readException();
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_result = _reply.readHashMap(cl);
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public String queryData(int flag, int what, String[] arg) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
String _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
mRemote.transact(Stub.TRANSACTION_queryData, _data, _reply, 0);
_reply.readException();
_result = _reply.readString();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public String queryDataArr(int flag, int what, String[] arg, int[] intarg) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
String _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
_data.writeIntArray(intarg);
mRemote.transact(Stub.TRANSACTION_queryDataArr, _data, _reply, 0);
_reply.readException();
_result = _reply.readString();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public String queryDataByMap(int flag, int what, String[] arg, int[] intarg, java.util.Map map) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
String _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
_data.writeIntArray(intarg);
_data.writeMap(map);
mRemote.transact(Stub.TRANSACTION_queryDataByMap, _data, _reply, 0);
_reply.readException();
_result = _reply.readString();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public java.util.Map queryData2Map(int flag, int what, String[] arg, int[] intarg) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
java.util.Map _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
_data.writeIntArray(intarg);
mRemote.transact(Stub.TRANSACTION_queryData2Map, _data, _reply, 0);
_reply.readException();
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_result = _reply.readHashMap(cl);
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public java.util.List queryData2List(int flag, int what, String[] arg, int[] intarg, boolean[] boolflag) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
java.util.List _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
_data.writeIntArray(intarg);
_data.writeBooleanArray(boolflag);
mRemote.transact(Stub.TRANSACTION_queryData2List, _data, _reply, 0);
_reply.readException();
ClassLoader cl = (ClassLoader)this.getClass().getClassLoader();
_result = _reply.readArrayList(cl);
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
@Override public boolean execBooleanAction(int flag, int what, String[] arg, java.util.Map arg1) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
boolean _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeInt(flag);
_data.writeInt(what);
_data.writeStringArray(arg);
_data.writeMap(arg1);
mRemote.transact(Stub.TRANSACTION_execBooleanAction, _data, _reply, 0);
_reply.readException();
_result = (0!=_reply.readInt());
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
}
static final int TRANSACTION_actionPerformed = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
static final int TRANSACTION_onReceiveMsg = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
static final int TRANSACTION_queryClientData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
static final int TRANSACTION_queryData = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
static final int TRANSACTION_queryDataArr = (android.os.IBinder.FIRST_CALL_TRANSACTION + 4);
static final int TRANSACTION_queryDataByMap = (android.os.IBinder.FIRST_CALL_TRANSACTION + 5);
static final int TRANSACTION_queryData2Map = (android.os.IBinder.FIRST_CALL_TRANSACTION + 6);
static final int TRANSACTION_queryData2List = (android.os.IBinder.FIRST_CALL_TRANSACTION + 7);
static final int TRANSACTION_execBooleanAction = (android.os.IBinder.FIRST_CALL_TRANSACTION + 8);
}
public void actionPerformed(int actionId) throws android.os.RemoteException;
//基本数据类型默认为in  can be an out type, so you must declare it as in, out or inout. 'out String flag2' can only be an in parameter.

public void onReceiveMsg(int flag, int what, boolean flag1, String flag2, java.util.Map map, java.util.List list) throws android.os.RemoteException;
public java.util.Map queryClientData(int flag, int waht, boolean flag1, String flag2, java.util.Map map, java.util.List list) throws android.os.RemoteException;
public String queryData(int flag, int what, String[] arg) throws android.os.RemoteException;
public String queryDataArr(int flag, int what, String[] arg, int[] intarg) throws android.os.RemoteException;
public String queryDataByMap(int flag, int what, String[] arg, int[] intarg, java.util.Map map) throws android.os.RemoteException;
public java.util.Map queryData2Map(int flag, int what, String[] arg, int[] intarg) throws android.os.RemoteException;
public java.util.List queryData2List(int flag, int what, String[] arg, int[] intarg, boolean[] boolflag) throws android.os.RemoteException;
public boolean execBooleanAction(int flag, int what, String[] arg, java.util.Map arg1) throws android.os.RemoteException;
}



```

### 绑定机器人服务

``` 
 // 本段代码在下面的onServiceConnected方法赋值返回
 public static RobotCallBinder mIRemoteService; 
// 下面代码实现方法在下面
public static ServiceConnection serviceConnection = new ServiceConnection() {
Intent intent = new Intent();
intent.setPackage("cn.qssq666.robot");
intent.setAction("cn.qssq666.robot.RemoteService");
 context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
```
### 绑定服务的serviceConnection实现类
``` 


public static ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.w(TAG, "onServiceConnected" + name + ",ibinder:" + service.getClass().getName());
            mIRemoteService = cn.qssq666.robot.RobotCallBinder.Stub.asInterface(service);
            try {
                mIRemoteService.registerCallback(new ICallBack.Stub() {
                    @Override
                    public void actionPerformed(int actionId) throws RemoteException {

                    }

                    @Override
                    public void onReceiveMsg(int flag, int what, boolean flag1, String flag2, Map map, List list) throws RemoteException {

                    }

                    @Override
                    public Map queryClientData(int flag, int waht, boolean flag1, String flag2, Map map, List list) throws RemoteException {
                        if (flag == ServiceExecCode.QUERY_USER_INFO) {
                         //TODO
                        } else if (flag == ServiceExecCode.QUERY_GROUP_INFO) {
                          //TODO
                        } else if (flag == ServiceExecCode.QUERY_LOGIN_INFO) {
                           //TODO
                        }
                        return null;
                    }
                    @Override
                    public String queryData(int flag, int what, String[] arg) throws RemoteException {
                        if (flag == ServiceExecCode.QUERY_NICKNAME) {
                            if (arg.length == 2) {
                                if (what == 0) {//查询昵称
                                    return QQEngine.queryNickName(arg[0], arg[1], what, -1);
                                } else {
                                    return QQEngine.queryGroupNickName(arg[0], arg[1]);

                                }

                            } else {
                                return "查询昵称不正确";
                            }
                        } else if (flag == ServiceExecCode.QUERY_GROUP_NAME) {//处理查询昵称的逻辑
                            return QQEngine.queryGrpupName(arg[0]);


                        } else if (flag == ServiceExecCode.QUERY_GROUP_INFO_FIELD) {
                            if (arg.length == 2) {

                                return QQEngine.queryGrpupInfoByField(arg[0], arg[1]);//处理查询群信息
                            } else {
                                return null;
                            }


                        } else if (flag == ServiceExecCode.QUERY_NICKNAME) {
                            if (arg.length == 2) {
                                if (what == 0) {
                                    return QQEngine.queryNickName(arg[0], arg[1], what, -1);//处理查询昵称
                                } else {
                                    return QQEngine.queryGroupNickName(arg[0], arg[1]);

                                }

                            } else {
                                return "查询昵称不正确";
                            }
                        } else if (flag == ServiceExecCode.QUERY_CURRENT_LOGIN_QQ) {//机器人那边发起查询登录账户的请求
                            return null;//TODO
                        } else if (flag == ServiceExecCode.QUERY_CURRENT_LOGIN_NICKNAME) {
                            return null;//TODO

                        }else if(flag==ServiceExecCode.REVOKE){//机器人那边发起的撤回消息逻辑
            //这里的arg 0 1 2 3可以从机器人源码那边得知结果
                        //  宿主的实现逻辑  return QQEngine.doRevokeMsg(arg[0],arg[1],arg[2],Long.parseLong(arg[3]));
                        }else if(flag==ServiceExecCode.CMD_GAG_USER){
                          //  TODO 宿主的实现逻辑    return QQEngine.requestGag(arg[0],arg[1],what);
                        }
                        return null;

                    }

                    @Override
                    public String queryDataArr(int flag, int what, String[] arg, int[] intarg) throws RemoteException {
                  //这里的逻辑可以灵活使用,不同的flag在这边 执行不同的操作

                        if (flag == ServiceExecCode.ADD_VOTE) {//点赞

                            int count = what;
                            String selfqq = arg[0];
                            String qq = arg[1];
                            if (BuildConfig.DEBUG) {
                                Log.w(TAG, "点赞 self qq" + selfqq + ",qq:" + qq);
                            }
                            int ignoreCheck = 0;
                            if (intarg != null && intarg.length > 0) {
                                ignoreCheck = intarg[0];
                            }
                               //  TODO 宿主的实现逻辑      String s = QQEngine.requestAddLike(selfqq, qq, count, ignoreCheck);
                              //  TODO 宿主的实现逻辑       return s;

                        }
//                             //  TODO 宿主的实现逻辑    String s = RemoteService.getClientICallBack().queryDataArr(ServiceExecCode.ADD_VOTE, count, new String[]{qq}, null);


                        return "unknowntype";
                    }

                    @Override
                    public String queryDataByMap(int flag, int what, String[] arg, int[] intarg, Map map) throws RemoteException {
                        return null;
                    }

                    @Override
                    public Map queryData2Map(int flag, int what, String[] arg, int[] intarg) throws RemoteException {

                        return null;
                    }

                    @Override
                    public List queryData2List(int flag, int what, String[] arg, int[] intarg, boolean[] boolflag) throws RemoteException {
                        return null;
                    }
                    @Override
                    public boolean execBooleanAction(int flag, int what, String[] arg, Map arg1) throws RemoteException {
                        if (flag == ServiceExecCode.CMD_KILL_QQ) {       //  TODO 宿主的实现逻辑  
                            Process.killProcess(Process.myPid());
                            System.exit(0);
                        }

                        return false;
                    }
                });
            } catch (RemoteException e) {
                writeLog("注册回调失败,将无法便捷的通过机器人查询宿主信息" + Log.getStackTraceString(e));
                e.printStackTrace();
            }


        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.w(TAG, "onServiceConnected ComponentName" + name);
            mIRemoteService = null;
            writeLog("机器人服务断开了");
        }
    }

```

下面是某+使用案例完整场景,切记不可利用下面代码进行二次发布,你的实现逻辑我这边不可能,从事违法活动本人概不负责,一切后果自行承担,本人目前也只是自弄自用.

下面的代码自行参考和集成无关,但是开发者可以根据下面代码找到一些实现方法的灵感技巧.

下面是上面提到的ServiceExecCode代码
``` 
package cn.qssq666.robot;
/**
 * Created by qssq on 2018/9/16 qssq521@gmail.com
 */
public interface ServiceExecCode {

    int QUERY_NICKNAME=1;
    int QUERY_CITY=2;
    int QUERY_ADDRESS=3;
    int QUERY_USER_INFO=4;
    int QUERY_GROUP_INFO=5;
    int QUERY_LOGIN_INFO=8;
    int CMD_KILL_QQ=6;

    int ADD_VOTE=7;//点赞
    int QUERY_CURRENT_LOGIN_QQ=9;
    int QUERY_CURRENT_LOGIN_NICKNAME=10;
    int QUERY_GROUP_NAME=11;
    int QUERY_GROUP_INFO_FIELD=12;
    int CMD_GAG_USER = 13;
    int REVOKE = 15;
}

```
RobotServiceUtil文件快速完成绑定判断.
首先在```application onCreate```
调用  ```RobotServiceUtil.startRobotService(context);```
如果本身是关闭状态的情况下而不是已启动就自动开启的情况,用户点击界面的开启绑定机器人服务逻辑
``` 
        switchItemWrap = new UiUtils.QQFormSwitchItemWrap(context);
        switchItemWrap.setText("绑定机器人服务");
        switchItemWrap.setChecked(QQConfigInner.robotAsServiceHostLoad && RobotServiceUtil.serviceIsLoad());
        switchItemWrap.setOnCheckedChangeListenerAndTagReturn(new UiUtils.IOnCheckListener<UiUtils.QQFormSwitchItemWrap>() {
            @Override
            public void onCheckedChanged(final UiUtils.QQFormSwitchItemWrap finalSwitchItemWrap, final boolean isChecked) {
                final UiUtils.IOnCheckListener that = this;
                QQConfigInner.updateRobotAsPluginLoad(context, false);
                if (!QQConfigInner.enableRobotAssist) {
                    if (!hasTipOpenRobot) {
                        hasTipOpenRobot = true;
                        Toast.makeText(context, "温馨提示，请勾选启用机器人哈,否则您只能在只能在机器人软件中测试查询昵称同步适配数据呢！", Toast.LENGTH_SHORT).show();

                    }
                }
                setCheckedFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_AS_PLUGIn_LOAD}, false);
                setOrHideDrawPluginEnterfaceItem(false, itemHashMap);

                int enableStartService = RobotServiceUtil.isEnableStartService(context);
                if (enableStartService != RobotServiceUtil.FLAG_QUERY_NORMAL) {

                    finalSwitchItemWrap.setOnCheckedChangeListener(null);
                    finalSwitchItemWrap.setChecked(false);
                    finalSwitchItemWrap.setOnCheckedChangeListenerAndTagReturn(this);
                    if (enableStartService == RobotServiceUtil.FLAG_QUERY_FAIL) {
                        UiUtils.checkAndRemoveCallBack(finalSwitchItemWrap, that, false);
                        Toast.makeText(context, "机器人软件必须先安装并保持运行才能开启此功能！", Toast.LENGTH_SHORT).show();
                        return;
                    }
                    if (enableStartService == RobotServiceUtil.FLAG_QUERY_NOT_SUPPORT) {
                        Toast.makeText(context, "机器人软件版本太低，至少版本code>=62方可启用此功能！", Toast.LENGTH_SHORT).show();
                        UiUtils.checkAndRemoveCallBack(finalSwitchItemWrap, that, false);
                    } else if (enableStartService == RobotServiceUtil.FLAG_QUERY_NOT_SUPPORT_ACTION) {
                        Toast.makeText(context, "没有找到相关机器人服务！", Toast.LENGTH_SHORT).show();
                        UiUtils.checkAndRemoveCallBack(finalSwitchItemWrap, that, false);
                    } else {
                        UiUtils.checkAndRemoveCallBack(finalSwitchItemWrap, that, false);

                        Toast.makeText(context, "无法启用,未知原因 flag=" + enableStartService, Toast.LENGTH_SHORT).show();

                    }
                } else {
                    if (isChecked) {
                        RobotServiceUtil.mError = "";
                        if (RobotServiceUtil.serviceIsLoad()) {
                            Object findObj = findViewById(itemHashMap, QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID);
                            if (findObj != null && findObj instanceof UiUtils.QQFormSimpleItemWrap) {
                                ((UiUtils.QQFormSimpleItemWrap) findObj).setRightText("机器人服务控制台");
                            }
                            Toast.makeText(context, "机器人服务已在运行！", Toast.LENGTH_SHORT).show();
                            setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID}, true);
                            QQConfigInner.updateRobotAsServiceHostLoad(context, true);
                        } else {
                            if (RobotServiceUtil.startRobotService(context.getApplicationContext())) {
                                final ProgressDialog progressDialog = DialogUtils.getProgressDialog((Activity) context);
                                progressDialog.setMessage("请稍后");
                                progressDialog.show();
                                new QssqTaskFix<Object, Object>(new QssqTaskFix.ICallBackImp() {
                                    @Override
                                    public Object onRunBackgroundThread(Object[] params) {
                                        int sleepCount = 0;
                                        try {
                                            while (true) {
                                                if (sleepCount > 8) {
                                                    return null;
                                                }
                                                if (RobotServiceUtil.serviceIsLoad()) {
                                                    return null;
                                                }

                                                Thread.sleep(500);
                                                sleepCount++;

                                            }
                                        } catch (InterruptedException e) {
                                            e.printStackTrace();
                                        }
                                        return null;
                                    }

                                    @Override
                                    public void onRunFinish(Object t) {
                                        progressDialog.dismiss();
                                        if (RobotServiceUtil.serviceIsLoad()) {
                                            setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_AS_PLUGIn_LOAD}, false);
                                            setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID}, true);
                                            Toast.makeText(context, "加载成功!", Toast.LENGTH_SHORT).show();
                                            QQConfigInner.updateRobotAsServiceHostLoad(context, true);
                                            Object findObj = findViewById(itemHashMap, QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID);
                                            if (findObj != null && findObj instanceof UiUtils.QQFormSimpleItemWrap) {
                                                ((UiUtils.QQFormSimpleItemWrap) findObj).setRightText("机器人服务控制台");
                                            }
                                        } else {
                                            setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_AS_PLUGIn_LOAD}, true);
                                            Toast.makeText(context, "绑定失败请尝试先终止情迁机器人和QQ程序->!" + RobotServiceUtil.mError, Toast.LENGTH_LONG).show();
                                            UiUtils.checkAndRemoveCallBack(finalSwitchItemWrap, that, false);


                                            setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID}, false);
                                            QQConfigInner.updateRobotAsServiceHostLoad(context, false);
                                        }

                                    }
                                }).execute();
                            }
                        }
                    } else {
                        boolean b = RobotServiceUtil.stopAndUnBindService(context);
                        setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_AS_PLUGIn_LOAD}, true);
                        setHideOrShowFromItemMap(itemHashMap, new String[]{QQConfigInner.KEY_ROBOT_SERVICE_HOST_ID}, false);
                        QQConfigInner.updateRobotAsServiceHostLoad(context, false);
                        Toast.makeText(context, "停止服务" + (b ? "成功" : "失败") + RobotServiceUtil.mError, Toast.LENGTH_LONG).show();

                    }

                }
            }
        });
        
        


```

上面的控制台代码,比如控制台里面有一个获取情迁机器人版本号的代码实际上是这样写的
```  
 try {
            StringBuffer sb = new StringBuffer();
            sb.append("版本名:" + RobotServiceUtil.mIRemoteService.getVersionName());
            sb.append("\n版本号:" + RobotServiceUtil.mIRemoteService.getVersionCode());
            if (RobotServiceUtil.mIRemoteService.getVersionCode() >= 90&&RobotServiceUtil.mIRemoteService.getVersionCode()<95) {
                sb.append("\n是否登录社区账号:" + RobotServiceUtil.mIRemoteService.isLogin());
                if (RobotServiceUtil.mIRemoteService.isLogin()) {
                    sb.append("\n登录用户:" + RobotServiceUtil.mIRemoteService.getLoginUser());
                    sb.append("\n授权Token:" + RobotServiceUtil.mIRemoteService.getLoginToken());
                }
            } else {
                sb.append("\n当前版本过低,不支持撤回用户消息");

            }
            DialogUtils.showDialog(activity, sb.toString());
        } catch (RemoteException e) {
            DialogUtils.showDialog(activity, "服务连接异常!");
        }

```
### 绑定机器人服务工具类完整案例代码

``` 

import android.app.Activity;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.ServiceConnection;
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.os.Handler;
import android.os.IBinder;
import android.os.Looper;
import android.os.Message;
import android.os.Process;
import android.os.RemoteException;
import android.support.annotation.NonNull;
import android.text.TextUtils;
import android.util.Log;

import org.json.JSONException;
import org.json.JSONObject;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.text.SimpleDateFormat;
import java.util.ArrayDeque;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import cn.qssq666.insertqqmodule.BuildConfig;
import cn.qssq666.robot.ServiceExecCode;
import corecode.a2.QQConfigInner;
import corecode.a2.QQEngine;
import cn.qssq666.robot.ICallBack;
import cn.qssq666.robot.RobotCallBinder;

import static corecode.a1.UiUtils.appIsExistForPackage;

/**
 * Created by qssq on 2018/9/16 qssq521@gmail.com
 */
public class RobotServiceUtil {

    private static final String TAG = "RobotServiceUtil";
    public static int FLAG_QUERY_FAIL = -1;
    public static int FLAG_QUERY_NOT_SUPPORT = -1;
    public static int FLAG_QUERY_NOT_SUPPORT_ACTION = -3;
    public static int FLAG_QUERY_NOT_RUNNNING = -2;
    public static int FLAG_QUERY_NORMAL = 0;
    static Handler handler = new Handler(Looper.getMainLooper(), new Handler.Callback() {
        @Override
        public boolean handleMessage(Message msg) {
            if (msg.what == 5) {
                writeLog("准备复活");


                if (!serviceIsLoad()) {

                    startRobotService(QQEngine.getContext());
                } else {
                    writeLog("无需复活");
                }
                return true;
            }
            return false;
        }
    });
    static int checkTime = 8 * 1000;
    static Runnable runnableAlive = new Runnable() {
        @Override
        public void run() {
            //如果是意外杀死，而不是通过手机抹掉进程，那么还是无法让他复活的。


            try {

                if (QQConfigInner.robotAsServiceHostLoad && QQConfigInner.robotAsServiceKeepAlive) {

                    if (mIRemoteService != null) {
                        int pid = mIRemoteService.getVersionCode();
                        writeLog("检测机器人服务为正常状态");
                        handler.postDelayed(this, checkTime);
                    } else {
                        writeLog("出现异常，服务控制器丢失,进行重新绑定");
                        startRobotService(QQEngine.getContext());
                        handler.postDelayed(this, 4500);
                    }


                } else {
                    writeLog("中断检测  是否绑定服务:" + QQConfigInner.robotAsServiceHostLoad + ",是否保持激活:" + QQConfigInner.robotAsServiceKeepAlive);
                }

            } catch (RemoteException e) {
                writeLog("检测机器人服务出现异常" + e.getMessage());
//                startRobotService(QQEngine.getContext());
                handler.sendEmptyMessageDelayed(5, 1000);
//                handler.postDelayed(this, 3000);

            }


        }
    };
    public static String mError;

    public static int isEnableStartService(Context context) {

        final PackageInfo qqrobotPackageInfo = appIsExistForPackage(context, QQConfigInner.qqrobotName);
        if (qqrobotPackageInfo == null) {
            return FLAG_QUERY_FAIL;
        } else {
            if (qqrobotPackageInfo.versionCode < 62) {
                return FLAG_QUERY_NOT_SUPPORT;
            } else {

                Intent robotServiceIntent = getRobotServiceIntent();
                List<ResolveInfo> resolveInfos = context.getPackageManager().queryIntentServices(robotServiceIntent, PackageManager.MATCH_ALL);
                if (resolveInfos == null || resolveInfos.isEmpty()) {
                    return FLAG_QUERY_NOT_SUPPORT;
                }
                return FLAG_QUERY_NORMAL;
            }
        }


    }


    public static boolean serviceIsLoad() {

        if (mIRemoteService == null) {
            return false;
        } else {

            try {
                if (mIRemoteService.isTaskRunning()) {

                }
                return true;
            } catch (RemoteException e) {
                e.printStackTrace();
                if (TextUtils.isEmpty(mError)) {
                    mError = Log.getStackTraceString(e);
                }
                writeLog("服务失效,没有加载 " + e.getMessage());


                return false;

            }
        }
    }

    public static RobotCallBinder mIRemoteService;
    public static ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.w(TAG, "onServiceConnected" + name + ",ibinder:" + service.getClass().getName());
            ;
            mIRemoteService = cn.qssq666.robot.RobotCallBinder.Stub.asInterface(service);


            try {
                mIRemoteService.registerCallback(new ICallBack.Stub() {
                    @Override
                    public void actionPerformed(int actionId) throws RemoteException {

                    }

                    @Override
                    public void onReceiveMsg(int flag, int what, boolean flag1, String flag2, Map map, List list) throws RemoteException {

                    }

                    @Override
                    public Map queryClientData(int flag, int waht, boolean flag1, String flag2, Map map, List list) throws RemoteException {
                        if (flag == ServiceExecCode.QUERY_USER_INFO) {

                            HashMap<String, String> mapquery = QQEngine.queryUserInfo(flag2);
                            return mapquery;
                        } else if (flag == ServiceExecCode.QUERY_GROUP_INFO) {

                            HashMap<String, String> mapquery = QQEngine.queryGroupInfo(flag2);
                            return mapquery;
                        } else if (flag == ServiceExecCode.QUERY_LOGIN_INFO) {

                            HashMap<String, String> mapquery = QQEngine.queryGroupInfo(flag2);
                            return mapquery;
                        }
                        return null;
                    }

                    @Override
                    public String queryData(int flag, int what, String[] arg) throws RemoteException {

                        if (flag == ServiceExecCode.QUERY_NICKNAME) {
                            if (arg.length == 2) {
                                if (what == 0) {
                                    return QQEngine.queryNickName(arg[0], arg[1], what, -1);
                                } else {
                                    return QQEngine.queryGroupNickName(arg[0], arg[1]);

                                }

                            } else {
                                return "查询昵称不正确";
                            }
                        } else if (flag == ServiceExecCode.QUERY_GROUP_NAME) {
                            return QQEngine.queryGrpupName(arg[0]);


                        } else if (flag == ServiceExecCode.QUERY_GROUP_INFO_FIELD) {
                            if (arg.length == 2) {

                                return QQEngine.queryGrpupInfoByField(arg[0], arg[1]);
                            } else {
                                return null;
                            }


                        } else if (flag == ServiceExecCode.QUERY_NICKNAME) {
                            if (arg.length == 2) {
                                if (what == 0) {
                                    return QQEngine.queryNickName(arg[0], arg[1], what, -1);
                                } else {
                                    return QQEngine.queryGroupNickName(arg[0], arg[1]);

                                }

                            } else {
                                return "查询昵称不正确";
                            }
                        } else if (flag == ServiceExecCode.QUERY_CURRENT_LOGIN_QQ) {


                            if (QQEngine.getQQAppInterface() != null) {
                                try {
                                    Method methodAccountUin = QQEngine.getQQAppInterface().getClass().getMethod("getCurrentAccountUin");
                                    return (String) methodAccountUin.invoke(QQEngine.getQQAppInterface());
                                } catch (NoSuchMethodException e) {
                                    e.printStackTrace();
                                } catch (IllegalAccessException e) {
                                    e.printStackTrace();
                                } catch (InvocationTargetException e) {
                                    e.printStackTrace();
                                }
                            }
                            return null;
                        } else if (flag == ServiceExecCode.QUERY_CURRENT_LOGIN_NICKNAME) {


                            if (QQEngine.getQQAppInterface() != null) {
                                try {
                                    Method methodAccountUin = QQEngine.getQQAppInterface().getClass().getMethod("getCurrentNickname");
                                    return (String) methodAccountUin.invoke(QQEngine.getQQAppInterface());
                                } catch (NoSuchMethodException e) {
                                    e.printStackTrace();
                                } catch (IllegalAccessException e) {
                                    e.printStackTrace();
                                } catch (InvocationTargetException e) {
                                    e.printStackTrace();
                                }
                            }
                            return null;

                        }else if(flag==ServiceExecCode.REVOKE){
                            return QQEngine.doRevokeMsg(arg[0],arg[1],arg[2],Long.parseLong(arg[3]));
                        }else if(flag==ServiceExecCode.CMD_GAG_USER){
                            return QQEngine.requestGag(arg[0],arg[1],what);
                        }
                        return null;

                    }

                    @Override
                    public String queryDataArr(int flag, int what, String[] arg, int[] intarg) throws RemoteException {


                        if (flag == ServiceExecCode.ADD_VOTE) {

                            int count = what;
                            String selfqq = arg[0];
                            String qq = arg[1];

                            if (BuildConfig.DEBUG) {
                                Log.w(TAG, "点赞 self qq" + selfqq + ",qq:" + qq);
                            }
                            int ignoreCheck = 0;
                            if (intarg != null && intarg.length > 0) {
                                ignoreCheck = intarg[0];
                            }
                            String s = QQEngine.requestAddLike(selfqq, qq, count, ignoreCheck);
                            return s;

                        }
//                        String s = RemoteService.getClientICallBack().queryDataArr(ServiceExecCode.ADD_VOTE, count, new String[]{qq}, null);


                        return "unknowntype";
                    }

                    @Override
                    public String queryDataByMap(int flag, int what, String[] arg, int[] intarg, Map map) throws RemoteException {
                        return null;
                    }

                    @Override
                    public Map queryData2Map(int flag, int what, String[] arg, int[] intarg) throws RemoteException {


                        return null;
                    }

                    @Override
                    public List queryData2List(int flag, int what, String[] arg, int[] intarg, boolean[] boolflag) throws RemoteException {
                        return null;
                    }

                    @Override
                    public boolean execBooleanAction(int flag, int what, String[] arg, Map arg1) throws RemoteException {
                        if (flag == ServiceExecCode.CMD_KILL_QQ) {
                            Process.killProcess(Process.myPid());
                            System.exit(0);
                        }

                        return false;
                    }
                });
            } catch (RemoteException e) {
                writeLog("注册回调失败,将无法便捷的通过机器人查询宿主信息" + Log.getStackTraceString(e));
                e.printStackTrace();
                if (TextUtils.isEmpty(mError)) {
                    mError = Log.getStackTraceString(e);
                }
            }
//                }

//            }, 3000);

        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.w(TAG, "onServiceConnected ComponentName" + name);
            mIRemoteService = null;

            writeLog("机器人服务断开了");

            if (QQConfigInner.robotAsServiceKeepAlive && QQConfigInner.robotAsServiceHostLoad) {
//                handler.sendEmptyMessageDelayed(5, 1000);


                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            writeLog("尝试复活机器人");
                startRobotService(QQEngine.getContext());

            } else {
                writeLog("没有勾选保持唤醒,因此放弃复活");

            }
            ;
        }
    };


    public static boolean startRobotService(Context context) {
        if (context instanceof Activity) {
            context = context.getApplicationContext();
        }

        try {
            mError = "";
            Intent intent = getRobotServiceIntent();

            context.bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
            writeLog("绑定服务成功");

            startDaemonCheck();
            return true;

        } catch (Exception e) {
            mError = Log.getStackTraceString(e);
            writeLog("绑定服务失败" + mError);
            stopAndUnBindService(context);

            return false;
        }
    }

    private static void startDaemonCheck() {
        handler.removeCallbacks(runnableAlive);
        handler.postDelayed(runnableAlive, checkTime);
    }

    @NonNull
    private static Intent getRobotServiceIntent() {
        Intent intent = new Intent();
        intent.setPackage("cn.qssq666.robot");
        intent.setAction("cn.qssq666.robot.RemoteService");
        return intent;
    }

    public static JSONObject buildResultInfoJson(String msg, int code) {
        JSONObject jsonObject = new JSONObject();
        try {
            jsonObject.put("msg", msg);
            jsonObject.put("code", code);
        } catch (JSONException e) {
            e.printStackTrace();
        }
        return jsonObject;
    }

    public static String getTime(long time) {
        return new SimpleDateFormat("yyyy:MM:dd:HH:mm:ss").format(new Date(time));
    }

    public static void writeLog(String str) {
        if (stack.size() >= 20) {
            stack.removeFirst();
        }
        stack.push("[" + getTime(System.currentTimeMillis()) + "]" + str);
    }

    public static ArrayDeque<String> stack = new ArrayDeque(20);

    public static boolean stopAndUnBindService(Context context) {
        if (mIRemoteService != null) {
            try {
                mIRemoteService.clearCallBack();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        mIRemoteService = null;
        try {
            if (context instanceof Activity) {
                context = context.getApplicationContext();
            }

            context.unbindService(serviceConnection);
            return true;
        } catch (Exception e) {
            if (TextUtils.isEmpty(mError)) {
                mError = "停止服务失败" + Log.getStackTraceString(e);
            }
            if (BuildConfig.DEBUG) {
                Log.w(TAG, e);
            }
        }
        return false;
    }

    public static String getLog() {
        StringBuilder sb = new StringBuilder();
        Iterator<String> stringIterator = stack.descendingIterator();
        while (stringIterator.hasNext()) {
            sb.append(stringIterator.next() + "\n");

        }

        return sb.toString();


    }

    public static String getRobotAccount() {
        if (RobotServiceUtil.serviceIsLoad()) {
            if (RobotServiceUtil.mIRemoteService != null) {
                try {
                    return RobotServiceUtil.mIRemoteService.getLoginUser();
                } catch (Throwable e) {
                }
            }
        } else {
            return "未绑定机器人";
        }
        return "未知";
    }
}




```
> 到此已经结束了,高级的使用方法也并不难,但是稳定性这个问题,需要自行在宿主这边处理

# 常见问题

## 发送没有反应
确保机器人是否在运行中,如果没有先打开,
确保是否绑定机器人,通过开发者工具测试模拟发送消息,看看是否走了正常的逻辑
调试运行

## 提升稳定性
由于机器人的特殊性,因此必须保持机器人稳定运行才能发挥他的价值,而且大部分时候人可能是和运行机器人的手机、或者电脑是不再同一个地方，那么稳定性就变得非常重要了,下面是我提供的几个建议
为了提高稳定性，和避免断网，可以尝试做如下操作
> 1.给手机设置锁屏永不断网
> 
> 2.给宿主,机器人添加后台白名单,通常在android 6.0会自动提示添加电量优化白名单
> 
> 3.禁止手机锁屏,这个可能就比较耗电了,不过这是提高稳定性的比较好的方法,不过我的手机本来屏幕坏了..
> 
> 4.使用电脑模拟器代替手机,电脑之所以稳定是因为电脑可以设置屏幕锁屏，但是模拟器程序不锁屏，不受环境等影响,而且选择4.4.夜神模拟器还是比较靠谱的,抢红包速度也是很快的。
> 
> 5.机器人软件开启死亡监控模式(检测到宿主如"QQ"被杀死后自动打开),检测到断网记录日志,等等。目前已经开发了一个 java机器人插件，会间隔10分钟就启动一次QQ.各位想下载的去更新入口找吧。
> 
> 6.弄一个配置稍微高一点的手机,因为有些低内存手机会反反复复的杀死后台的机器人程序,搞得体验很不好,比如红米4a换成红米note4x之后稳定性强多了,每天都在线，天天挂机
>
> 7.开发者在宿主app中编写定时启动唤醒机器人的代码.


## 插件不生效
查看插件是否加载出现错误,可以利用机器人里面的测试运行来测试规范.

## 调试插件
直接打开机器人插件源码直接给```cn.qssq666.robot```断点调试就行.

# 愿景
## 界面完善
可把收到的消息在插件界面这边进行展示,实现宿主和本机器人同时查看消息,以及更便捷的测试消息
## 自动升级
不想弄，需要烧钱少服务器.

## 插件生态
同样需要服务器运转.

## 增加root版控制能力
同样没时间没精力弄,只有在某些激情来的时候才会倒腾新增一两个功能
# 源代码下载
https://github.com/qssq/robot

# 下载升级

由于作者对应的宿主支持app不再发布,因此机器人的发布意义不是很大,不过我依然会进行发布,另外我基本上不会上传到本网站以及博客网站中,常见的更新地址如下：
qq群， telegram(需要翻墙)
robot 就是机器人的意思，别进了群都不知道怎么下载哈!
*不可使用本软件进行诈骗从事违法行为等行为，仅供学习使用！触发法律后果自负！本人不承担任何责任，软件使用的时候协议上也一再强调禁止用于违法用途！切记切记*

群主非本人
[qq群 748171411](https://jq.qq.com/?_wv=1027&k=voeyhUxG)


qq 35068264的qq空间可以翻找说说，我有时会会把加群图片放在QQ空间

电报 : https://t.me/qssq666
# 学习更多机器人的相关知识以及机器人插件开发的相关教程
https://www.jianshu.com/nb/24436393

## 联系我

qq:35068264

email:qssq521@gmail.com

电报 : https://t.me/qssq666



![qq35068264](pic/wechat.png)

![qq35068264](pic/qq.png)



# 打赏赞助

alipay:qssq521@gmail.com

qq:35068264
 

