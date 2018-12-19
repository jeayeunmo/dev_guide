
# Session management supportig document

This document was made as Frank Kim (Principal Solution Architect)' request.

Libraries and reference site included:
- [JWT for Android](https://github.com/auth0/JWTDecode.Android)
- [JWT(JSON Web Tokens)](https://jwt.io/)

## History
- Maple uses Okta for SSO(single sign-on) and receive some tokens(access_token, refresh_token, id_token) from Okta.
- Maple app uses those tokens to access  backend system(digital pharmacy, qhr system)
- If id_token is expired, it is possible to renew that token using refresh_token.

## Requirements
- Maple App checks if access_token is expired or not, as the result it renew the tokens

### How to implement it

1. add JWT library - build.gradle(Module: MapleApplication)
```xml
    //JWTDecode.Android
    implementation ('com.auth0.android:jwtdecode:1.2.0'){
        exclude group: 'com.android.support', module: 'appcompat-v7'
    }
```

2. add a new endpoint to get the new tokens - OktaService.java
```java

    // OAuth2 get new token by refresh token
    @FormUrlEncoded
    @POST("oauth2/v1/token")
    Call<TokenResponseModel> doGetNewTokens(@Field("client_id") String client_id, @Field("grant_type") String grant_type,
    @Field("scope") String scope, @Field("refresh_token") String refresh_token, @Field("redirect_uri") String redirect_uri);
```

3. add new variable to save the tokens - ApiController.java
```java

    private TokenResponseModel mTokenResponseModel;
    
    //get
    public TokenResponseModel getTokenResponseModel() {
        return mTokenResponseModel;
    }

    //set
    public void setTokenResponseModel(TokenResponseModel tokenResponseModel) {
        mTokenResponseModel = tokenResponseModel;
    }

```

4. add mApi.setTokenResponseModel to save the tokens - AuthenticationRepository.java
```java

       public LiveData<Resource<TokenResponseModel>> observeTokens() {
        mTokenObserver = primaryAuthModel -> {
            if (primaryAuthModel == null) return;
            mTokens = primaryAuthModel.body;
            mApi.setOktaIdToken(primaryAuthModel.body.id_token);

            //[add]setTokenResponseModel
            mApi.setTokenResponseModel(primaryAuthModel.body);

        };

        return mTokenResponse;
    }

```

5.add check logic and renew token logic - BaseRepository.java
5.1 add check logic
```java
   protected <T> void genericCall(MediatorLiveData<Resource<T>> mediator,
            LiveData<ApiResponse<T>> retrofitCall, Observer<ApiResponse<T>> onChanged,
            Class<T> clazz) {
        
        mediator.setValue(Resource.loading(null));

        /**
         * [Add]
         * 1. check where TokenResponseModel has a value
         * 2. expiration check of id_token
         * 3. if id_token is expired, call renewal method
         */
        TokenResponseModel tokenResponseModel = mApi.getTokenResponseModel();
        
        if (tokenResponseModel != null && !TextUtils.isEmpty(tokenResponseModel.id_token)) {
            try {
                JWT jwt = new JWT(tokenResponseModel.id_token);
                boolean isExpired = jwt.isExpired(tokenResponseModel.expires_in);

                if (isExpired) {
                  renewOktaToken(tokenResponseModel);
                }
                
            } catch (DecodeException e) {
                Log.e(TAG, "DecodeException : " + e.getMessage());
            }

        }

      mediator.addSource(retrofitCall, apiResponse -> {
      ......
```

5.2 add renew token logic
```java
   private void renewOktaToken(TokenResponseModel tokenResponseModel) {
        String grant_type = "refresh_token";
        String client_id = BuildConfig.OKTA_CLIENT_ID;
        String redirect_rui = "pcv://login/appredirect"; //from AuthenticationRepository
        String refresh_token = tokenResponseModel.refresh_token;
        String scope = tokenResponseModel.scope;

        genericCallForNewToken(client_id, grant_type, scope, refresh_token,
                redirect_rui,tokenResponseModel);
    }
```

5.3 add retrofit2 logic (http call)
```java

     protected void genericCallForNewToken(String client_id, String grant_type, String scope,
            String refresh_token, String redirect_rui, TokenResponseModel tokenResponseModel) {

        retrofit2.Call<TokenResponseModel> call = mApi.getOktaService().doGetNewTokens
                (client_id, grant_type, scope, refresh_token, redirect_rui);

        call.enqueue(new retrofit2.Callback<TokenResponseModel>() {
            @Override
            public void onResponse(retrofit2.Call<TokenResponseModel> call,
                    retrofit2.Response<TokenResponseModel> response) {
                if (response.isSuccessful() && response.body() != null) {

                    tokenResponseModel.scope = response.body().scope;
                    tokenResponseModel.token_type = response.body().token_type;
                    tokenResponseModel.access_token = response.body().access_token;
                    tokenResponseModel.refresh_token = response.body().refresh_token;
                    tokenResponseModel.id_token = response.body().id_token;
                    tokenResponseModel.expires_in = response.body().expires_in;

                    //update tokens
                    mApi.setOktaIdToken(tokenResponseModel.id_token);
                    mApi.setTokenResponseModel(tokenResponseModel);

                } else {
                    Log.e(TAG, "genericCallForNewToken reponse is fail or null");
                }
            }

            @Override
            public void onFailure(retrofit2.Call<TokenResponseModel> call, Throwable t) {
                Log.e(TAG, "genericCallForNewToken onFailure :" + t.getMessage());
            }
        });


    }
```


6. Result.
 - before
 before token id_token: eyJraWQiOiJVM0lKckZleFFLUHA3aUZ2dmVrZ3hHMzA5QTBWU2prMHpHNXVUNFJjUVVvIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHVnY3A0aDQ1YTRsYXBpYTBoNyIsIm5hbWUiOiJqZWF5ZXVuLm1vQHFocnRlY2guY29tIiwiZW1haWwiOiJqZWF5ZXVuLm1vQHFocnRlY2guY29tIiwidmVyIjoxLCJpc3MiOiJodHRwczovL3NhbmRib3g0MGgub2t0YXByZXZpZXcuY29tL29hdXRoMi9hdXNmb283c2RkOFVmT2tCbjBoNyIsImF1ZCI6IjBvYWZyOHBrZzVZNFpNUDJWMGg3IiwiaWF0IjoxNTQ1MjUyNTk5LCJleHAiOjE1NDUyNTYxOTksImp0aSI6IklELlNEendpUGxadHdFbDFidWVSRVhBaUdGRThqM0gzZHNmaGZXWkR5MndPUmsiLCJhbXIiOlsicHdkIl0sImlkcCI6IjAwb2Yxb2V0bGVxc2FxSmZqMGg3IiwicHJlZmVycmVkX3VzZXJuYW1lIjoiamVheWV1bi5tb0BxaHJ0ZWNoLmNvbSIsImF1dGhfdGltZSI6MTU0NTI1MjU5NiwiYXRfaGFzaCI6InJBWWt4UjVyX0Z0R0Rua1RnZ21xenciLCJtZmEiOiJubyIsInBjaWQiOiI2ZTI0OWNiNC00MGJkLTRkNTItYjFjNy03MmRlNzMxMDRmNzEifQ.se4m14Lc6GHPo1sd9RD08c42xRwjiNgYgsdARIKZSrDMFmrHnENcyX9wPz4QBOLLNLafZ_yhzJTy2o3aTBRqwVIxU3rBTyk9mXSd5FFbxvt7rokZoYp5xKmjyJecsEfr1cQqZD4q3YI2FMpt6oPooTB0P_UvZ8_npA5cVBvN1w1DqeiCXoL6sND8DUPu_4WvAVXie1yS6Umzb2ONn_F5tXsgkqFwm2vdmvxcU9zgwik9FKfAiQENYNs4U9tbVN5EgtJsQcaprq0V7fm95jidfz20yJmP4u3Psm1RB941oo7ED5btfxJR_y54syu3ono-OOASa2enqpmAsf4q8NR_1Q
 
 - after
 new token id_token: eyJraWQiOiJVM0lKckZleFFLUHA3aUZ2dmVrZ3hHMzA5QTBWU2prMHpHNXVUNFJjUVVvIiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHVnY3A0aDQ1YTRsYXBpYTBoNyIsIm5hbWUiOiJqZWF5ZXVuLm1vQHFocnRlY2guY29tIiwiZW1haWwiOiJqZWF5ZXVuLm1vQHFocnRlY2guY29tIiwidmVyIjoxLCJpc3MiOiJodHRwczovL3NhbmRib3g0MGgub2t0YXByZXZpZXcuY29tL29hdXRoMi9hdXNmb283c2RkOFVmT2tCbjBoNyIsImF1ZCI6IjBvYWZyOHBrZzVZNFpNUDJWMGg3IiwiaWF0IjoxNTQ1MjUyNjAxLCJleHAiOjE1NDUyNTYyMDEsImp0aSI6IklELlVqS3EzdExaWlppdTZoQlVfRmduSU91Q3NBb09xZy01R2hRWjNrZjFTaGsiLCJhbXIiOlsicHdkIl0sImlkcCI6IjAwb2Yxb2V0bGVxc2FxSmZqMGg3IiwicHJlZmVycmVkX3VzZXJuYW1lIjoiamVheWV1bi5tb0BxaHJ0ZWNoLmNvbSIsImF1dGhfdGltZSI6MTU0NTI1MjU5NiwiYXRfaGFzaCI6Ik4wS1d1djlrOWFNWFJNcV80MHdkU1EiLCJtZmEiOiJubyIsInBjaWQiOiI2ZTI0OWNiNC00MGJkLTRkNTItYjFjNy03MmRlNzMxMDRmNzEifQ.fDgFNKh9NUvh1xKXuOuK0xEu8jNGfULBerVZPS_BZHn9kCsLwUtf1XkIRbkGxkxFxKmD0Mb41Q3AZ9dJrjVsINVMdzAWy_LnLLxB9o9bFjeoav7rzQqrOaiuzWeXDxsNJaQ1HyvaZiXFbJpGG4NNkRzh0RFI74HjM0W65TkjDQR60bJaPbJEmnlUYUB514kuYeb2yYR13hUYrD7FyvB5t203eEFEAGfL0ZND19W2MqVo4ZE0zwRRTl64QUEmjsfswTLDLi_TQc0ZCLuujUg5JcW5U-vFUboZFuiqFFT9Dl9NeMHBagXUQ_F7lGU56DNdQmOfquUrX2XVxMHQfFmG0w


 



