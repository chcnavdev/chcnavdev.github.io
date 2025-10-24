Before connecting with CORS, you need to prepare the account number, password, source, and corresponding IP address and port number used for CORS connection. The specific connection code is as follows:

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

