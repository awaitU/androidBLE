# 一，获取开发所需的权限  
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />

# 二，发现设备  
        //获取蓝牙适配器
        final BluetoothManager bluetoothManager = (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
        BluetoothAdapter mBluetoothAdapter = bluetoothManager.getAdapter();
       // 设置搜索设备时间，到时停止搜索设备
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    mScanning = false;
                    mBluetoothAdapter.stopLeScan(mLeScanCallback);
                    cancelButton.setText(R.string.scan);
                }
            }, SCAN_PERIOD);
            mScanning = true;
            mBluetoothAdapter.startLeScan(mLeScanCallback);

 //回调函数，添加设备address到列表显示  
private BluetoothAdapter.LeScanCallback mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
        @Override
        public void onLeScan(final BluetoothDevice device, final int rssi,
                             byte[] scanRecord) {
            runOnUiThread(new Runnable() {
                @Override
                public void run() {

                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            addDevice(device, rssi);
                        }
                    });
                }
            });
        }
    };
    private void addDevice(BluetoothDevice device, int rssi) {
        boolean deviceFound = false;
        String devName = device.getName();
        if (TextUtils.isEmpty(devName) || !(devName.startsWith("Pace Band") || devName.startsWith("HUASHAN"))) {
            return;
        }
        for (BluetoothDevice listDev : deviceList) {
            if (listDev.getAddress().equals(device.getAddress())) {
                deviceFound = true;
                break;
            }
        }
        devRssiValues.put(device.getAddress(), rssi);
        if (!deviceFound) {
            deviceList.add(device);
            mEmptyList.setVisibility(View.GONE);

            deviceAdapter.notifyDataSetChanged();
        }
    }

# 三，连接设备  
为设备列表添加点击事件，点击连接设备：
注意：连接设备的时候需要停止搜索！
核心代码：
回传蓝牙地址信息到原活动页面
 public void onItemClick(AdapterView<?> parent, View view, int position,
                                long id) {
            BluetoothDevice device = deviceList.get(position);
            mBluetoothAdapter.stopLeScan(mLeScanCallback);
            Bundle b = new Bundle();
            b.putString(BluetoothDevice.EXTRA_DEVICE, deviceList.get(position)
                    .getAddress());
            Intent result = new Intent();
            result.putExtras(b);
            setResult(Activity.RESULT_OK, result);
            finish();

        }
//接收信息，开始连接
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == 888888 && resultCode == RESULT_OK) {
            onDeviceSelect(data);
        }
        super.onActivityResult(requestCode, resultCode, data);
    }

    private void onDeviceSelect(Intent data) {
        String deviceAddress = data
                .getStringExtra(BluetoothDevice.EXTRA_DEVICE);
        BluetoothDevice mDevice = BluetoothAdapter.getDefaultAdapter().getRemoteDevice(
                deviceAddress);

        Log.d(TAG, "onDeviceSelect.address: " + mDevice);
        appendText("尝试连接设备" + deviceAddress);
        mService.connect(deviceAddress);
    }

//此为connect函数核心代码,其中最为重要的就是mGattCallback函数，里面包含连接状态监控，服务发现以及接收数据
       final BluetoothDevice device = mBluetoothAdapter
                .getRemoteDevice(address);
       mBluetoothGatt = device.connectGatt(this.mContext, false, mGattCallback);

//下面是mGattCallback的代码
private BluetoothGattCallback mGattCallback = new BluetoothGattCallback() {
        public void onConnectionStateChange(BluetoothGatt gatt, int status,
                                            int newState) {

            if (newState == BluetoothProfile.STATE_CONNECTED) {
                mConnectionState = STATE_CONNECTED;
                doLog(TAG, "Connected to GATT server.");
                // Attempts to discover services after successful connection.
                doLog(TAG, "Attempting to start service discovery:"
                        + mBluetoothGatt.discoverServices());
                synchronized (this) {
                    if (mListener != null) {
                        mListener.onDeviceConnect(mBluetoothDeviceAddress);
                    }
                }
            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
                mBluetoothGatt.close();
                mBluetoothGatt = null;
                mConnectionState = STATE_DISCONNECTED;
                mServiceDiscovered = false;
                doLog(TAG, "Disconnected from GATT server.");
                if (mListener != null) {
                    mListener.onDeviceDisconnect(mBluetoothDeviceAddress);
                }
                tryReConnect();
            }
        }

        public void enableTXNotification() {
            BluetoothGattService uartService_nordic = mBluetoothGatt.getService(RX_SERVICE_UUID_NORDIC);
            BluetoothGattService uartService_dialog = mBluetoothGatt.getService(RX_SERVICE_UUID_DIALOG);
            if (uartService_nordic == null) {
                showMessage("NORDIC uart service not found!");
            } else { 
                BluetoothGattCharacteristic TxChar = uartService_nordic.getCharacteristic(TX_CHAR_UUID_NORDIC);
                if (TxChar == null) {
                    showMessage("Nordic Tx charateristic not found!");
                    return;
                }
                mBluetoothGatt.setCharacteristicNotification(TxChar, true);
                BluetoothGattDescriptor descriptor = TxChar.getDescriptor(CCCD);
                descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
                mBluetoothGatt.writeDescriptor(descriptor);
            }

            if (uartService_dialog == null) {
                showMessage("Dialog uart service not found!");
            } else {
                BluetoothGattCharacteristic TxChar = uartService_dialog.getCharacteristic(TX_CHAR_UUID_DIALOG);
                if (TxChar == null) {
                    showMessage("Dialog Tx charateristic not found!");
                    return;
                }
                mBluetoothGatt.setCharacteristicNotification(TxChar, true);
                BluetoothGattDescriptor descriptor = TxChar.getDescriptor(CCCD);
                descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
                mBluetoothGatt.writeDescriptor(descriptor);
            }
        }
        @Override
        public void onServicesDiscovered(BluetoothGatt gatt, int status) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                mServiceDiscovered = true;
                doLog(TAG, "onServicesDiscovered mBluetoothGatt = " + mBluetoothGatt);
                enableTXNotification();
            } else {
                doLog(TAG, "onServicesDiscovered received: " + status);
            }
        }

        @Override
        public void onCharacteristicRead(BluetoothGatt gatt,
                                         BluetoothGattCharacteristic characteristic, int status) {
            doLog(TAG, "onCharacteristicRead: " + gatt);
            if (status == BluetoothGatt.GATT_SUCCESS) {
                // onDataRecieved(characteristic);
            }
        }

        @Override
        public void onCharacteristicChanged(BluetoothGatt gatt,
                                            BluetoothGattCharacteristic characteristic) {
            //doLog(TAG, "onCharacteristicChanged: " + gatt);
            onDataRecieved(characteristic);
        }
    };


# 四， 数据通信  
两端要进行数据通信的话，肯定需要桥梁，这个桥梁就是BLE中独特的service与characteristic，根据约定的service与characteristic值（在上面连接中回调函数可以获取所有的服务与特征值，一般厂家会给出所需值，自己测试的话需要尝试获取指定值）进行数据通信。

//发送数据的核心代码：
        BluetoothGattService UartService_Dialog = mBluetoothGatt.getService(RX_SERVICE_UUID_DIALOG);
        BluetoothGattCharacteristic RxChar = UartService_Dialog.getCharacteristic(RX_CHAR_UUID_DIALOG);
        RxChar.setValue(value);
        boolean status = mBluetoothGatt.writeCharacteristic(RxChar);

# 五， 总结  
android BLE蓝牙开发的总流程为：获取权限→搜索设备→连接设备→数据通信
参考：
Android 蓝牙4.0（BLE）开发实现对蓝牙的写入数据和读取数据：
https://blog.csdn.net/HAndroidevelopcker/article/details/78614508
android蓝牙连接通信的实现：
https://blog.csdn.net/taoyuxin1314/article/details/78928249














