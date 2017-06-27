##Android开发之蓝牙连接与配对设备

Android平台包含蓝牙网络支持,设备能以无线方式与其他蓝牙设备交换数据。应用框架提供了通过 Android Bluetooth API 访问蓝牙功能的途径。 这些 API 允许应用以无线方式连接到其他蓝牙设备，从而实现点到点和多点无线功能。

下面这张图就是一般蓝牙的示意图。

![](http://i.imgur.com/6h9fLoE.jpg)

这里有几个概念：

-  UUID 是一串数字，两个蓝牙设备之前通过相同的UUID进行连接，我们可以理解为是类似端口的东西
-  BlueToothDexice 蓝牙设备的抽象对象，里面封装了蓝牙设备的名称，地址，是否已经配对过等信息
-  BlueToothSocket 蓝牙设备之间的通道，类似于网络通信的socket。能够提供输入与输出流。两个设备通过这个进行交互



###一、配置蓝牙权限
	<!--允许程序连接到已配对的蓝牙设备--!>
	<uses-permission android:name="android.permission.BLUETOOTH" />
	<!--允许程序发现和配对蓝牙设备--!>
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
    <!--允许程序获取位置--!>
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
**注意：当系统大于6.0时 开启蓝牙需要动态获取位置服务权限**

在onCreate中判断当前是否大于6.0
	
	if (Build.VERSION.SDK_INT >= 23) {
      	ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.ACCESS_FINE_LOCATION},0);
   	}
   	
 

动态权限配置回调

   	public void onRequestPermissionsResult(int requestCode, String permissions[], int[] grantResults) {
        switch (requestCode) {
            case 0: {
                // If request is cancelled, the result arrays are empty.
                if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                    Log.i("main", "添加权限");
                    //permission granted!
                    //add some code
                }
                return;
            }
        }
    }
    
###二、开启蓝牙
获取BluetoothAdapter蓝牙适配器实例

	mBluetoothAdapter = BluetoothAdapter.getDefaultAdapter();

通过startActivityForResult()方法发起开启蓝牙，根据常量值ACTION\_REQUEST\_ENABLE
	
   	/**如果本地蓝牙没有开启，则开启*/
   	if (!mBluetoothAdapter.isEnabled()) {
    	// 我们通过startActivityForResult()方法发起的Intent将会在onActivityResult()回调方法中获取用户的选择，比如用户单击了Yes开启，
        // 那么将会收到RESULT_OK的结果，
        // 如果RESULT_CANCELED则代表用户不愿意开启蓝牙
        Intent mIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
        startActivityForResult(mIntent, ENABLE_BLUE);
  	} else {
        Toast.makeText(this, "蓝牙已开启", Toast.LENGTH_SHORT).show();
   	}
在onActivityResult()判断蓝牙是否开启成功（ENABLE_BLUE是自己定义的常量值）

	@Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == ENABLE_BLUE) {
            if (resultCode == RESULT_OK) {
                Toast.makeText(this, "蓝牙开启成功", Toast.LENGTH_SHORT).show();
                getBondedDevices();
            } else if (resultCode == RESULT_CANCELED) {
                Toast.makeText(this, "蓝牙开始失败", Toast.LENGTH_SHORT).show();
            }
        } else {

        }
    }
    
###三、设置蓝牙可见
根据常量值BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION，通过Intent开启广播的方式设置开启蓝牙可见。

	Intent intent = new Intent(BluetoothAdapter.ACTION_REQUEST_DISCOVERABLE);
    intent.putExtra(BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION, 180);//180可见时间 单位s
    startActivity(intent);
###四、搜索蓝牙
搜索蓝牙是根据注册广播接收者获取到搜索到的蓝牙设备信息。

相关常量值：BluetoothDevice.ACTION_FOUND、BluetoothAdapter.ACTION_DISCOVERY_FINISHED

BluetoothDevice是Android提供的蓝牙设备类。

1.创建广播接收者（BlueDevice是自定义的bean类）

	private BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();


            /** 搜索到的蓝牙设备*/
            if (BluetoothDevice.ACTION_FOUND.equals(action)) {
                BluetoothDevice device = intent
                        .getParcelableExtra(uetoothDevice.EXTRA_DEVICE);
                // 搜索到的不是已经配对的蓝牙设备
                
                    BlueDevice blueDevice = new BlueDevice();
                    blueDevice.setName(device.getName() == null ? device.getAddress() : device.getName());
                    blueDevice.setAddress(device.getAddress());
                    blueDevice.setDevice(device);
                    setDevices.add(blueDevice);
                    blueAdapter.setSetDevices(setDevices);
                    blueAdapter.notifyDataSetChanged();
                    Log.d(MAINACTIVITY, "搜索结果......" + device.getName() + device.getAddress());
                
                /**当绑定的状态改变时*/
            } else if (action.equals(BluetoothAdapter.ACTION_DISCOVERY_FINISHED)) {
                setProgressBarIndeterminateVisibility(false);
                Log.d(MAINACTIVITY, "搜索完成......");
                hideProgressDailog();
            }
        }
    };
    
2.注册广播接收

 	mFilter = new IntentFilter(BluetoothDevice.ACTION_FOUND);
    mFilter.addAction(BluetoothDevice.ACTION_BOND_STATE_CHANGED);    //绑定状态监听
    mFilter.addAction(BluetoothAdapter.ACTION_DISCOVERY_FINISHED);     //搜索完成时监听
    registerReceiver(mReceiver, mFilter);
 
3.开启蓝牙搜索

	showProgressDailog();
    // 如果正在搜索，就先取消搜索
    if (mBluetoothAdapter.isDiscovering()) {
    	mBluetoothAdapter.cancelDiscovery();
    }
    Log.i("main", "我在搜索");
    // 开始搜索蓝牙设备,搜索到的蓝牙设备通过广播接受者返回
    mBluetoothAdapter.startDiscovery();
 
###五、配对蓝牙设备
 
两个蓝牙设备连接，需要以下的两种类型的设备存在,一种是服务端，一种是客户端。

![](http://i.imgur.com/ohyI0lz.png)

编写服务端代码

首先需要生成配对的UUid,生成的方法有下面三种：

- 1 在window下面，可以使用http://www.uuid.online/网站生成
- 2 在mac平台下面，在命令行使用uuidgen命令生成
- 3 使用java Api 生成.UUID.randomUUID()        


        <uses-permission android:name="android.permission.BLUETOOTH"/>
        <uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/> 
        
 注册需要使用的权限。注意在6.0版本的手机上面使用，需要动态申请权限。
 
 ![](http://i.imgur.com/XnzHRf3.png)

 ![](http://i.imgur.com/J2DSexq.png) 
   
 ![](http://i.imgur.com/GHJDiz6.png)  
 
 ![](http://i.imgur.com/cHqMI92.png)  
 
 ![](http://i.imgur.com/7mYsoZa.png) 
 
 ![](http://i.imgur.com/Ka2Vd9A.png)  
 
 ![](http://i.imgur.com/HPqKkhw.png)  
 
 ![](http://i.imgur.com/4aiDvQb.png) 

接着我们编写客户端

 ![](http://i.imgur.com/6mPI6Je.png)  
 
 ![](http://i.imgur.com/z4vkYcl.png) 

 ![](http://i.imgur.com/w57BGZR.png)  

 ![](http://i.imgur.com/wFlGXiA.png) 

 ![](http://i.imgur.com/HI25nh0.png)  

 ![](http://i.imgur.com/Q7QpRgS.png)  


操作过程至此结束。
