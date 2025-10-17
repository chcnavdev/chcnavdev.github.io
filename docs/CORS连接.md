CORS连接之前，您需要准备用于CORS连接的账号、密码、源以及对应的IP地址和端口号，具体的连接代码如下：

```java
mCorsConnect.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                DiffDataInfo diffDataInfo = new DiffDataInfo();
                // ip, port
                AddressInfo addressInfo = new AddressInfo("*.*.*.*", *);    
                diffDataInfo.setAddressInfo(addressInfo);
                diffDataInfo.setDiffType("diffType");
                // source
                diffDataInfo.setSourcePoint("***");
                //account
                diffDataInfo.setUserName("****"); 
                // password    
                diffDataInfo.setPassWord("****");     
                // setting listener
                DiffConnectManager.getInstance().addListener(new IDiffListener() {
                    @Override
                    public void onConnectStatusChanged(boolean b, @NonNull NtripProtocolCode ntripProtocolCode) {
                       .......
                    }

                    @Override
                    public void onPdaWorkModeStatusChanged(@NonNull EmPdaWorkModeStatus emPdaWorkModeStatus) {
                        .......
                    }
                });

                DiffConnectManager.getInstance().connect(diffDataInfo);
            }
        });

        mCorsDisConnect.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                DiffConnectManager.getInstance().disConnect();
            }
        });

```

