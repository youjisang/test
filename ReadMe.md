# SkillShare 사용 기술 stack

### 사용 Library

#### Layout 및 뷰 관련 library

- cardview
- bottom navigation
- togglebutton
- flexboxLayout

#### network 관련 library

- retrofit2
    - gsonconverter, adapter-rxjava

- okhttp

#### 동영상 관련 library

- exo player

#### LogIn 관련 library

- google login

#### notification 관련 library

- google cloud messaging

#### image loading 관련 library

- glide

#### Reactive 관련 library

- rxandroid2
- rxbinding2

#### 기타

- lambda

## Application structure

- 사진첨부

# Application analysis


## Splash Activity

### _구조 및 기능_

![splash](https://user-images.githubusercontent.com/31605792/35350273-c9a74c9a-0180-11e8-84cb-5e1cd7c337f1.png)
- 최초 app 화면으로, 앱의 진입점을 담당하는 Activity

### _issue_

- 자동로그인 관련 처리
- gif파일 화면세팅 처리

### _How to solve?_

> __Preference로 자동로그인 처리__

- 정상적으로 SignUp,SignIn이 이루어지면 해당 Token값이 Preference에 저장이 되고, 나중에 app을 켤때는 __Preference에 저장되어 있는 값을 다음과 같이 추출하는 과정__을 통해 자동으로 로그인을 할 수 있는 정보가 세팅이 된다.

```Java
String token = PreferenceUtil.getStringValue(this, ConstantUtil.AUTHORIZATION_FLAG);
```

- 이하 세팅된 token 값을 활용한 서버 통신 과정.

```Java
if(!ValidationUtil.isEmpty(token)) {
            RetrofitHelper.createApi(UserService.class)
                    .getMyInfo(token)
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(
                            (UserResponse userResponse) -> {
                                if(ConstantUtil.SUCCESS.equals(userResponse.getResult())) {
                                    StateUtil.getInstance().setUserInstance(userResponse.getData());
                                    Intent intent = new Intent(SplashActivity.this, MainActivity.class);
                                    intent.setAction(ConstantUtil.SIGN_IN_SUCCESS);
                                    startActivity(intent);
                                    finish();
                                } else {
                                    Toast.makeText(this, userResponse.getMessage(), Toast.LENGTH_LONG).show();
                                }
                            }, (Throwable error) -> {

                            }
                    );
        } else {
            setContentView(R.layout.activity_splash);
```

> __Glide를 사용하여 gif파일 화면 세팅처리__

- glide library를 활용하면 쉽게 gif파일을 세팅할 수 있다.
- gif 프레임 추출 > 프레임 사이즈 최적화 > 프레임 저장

```java
  
            Glide.with(this)
                    .load(R.raw.splash)
                    .apply(RequestOptions.diskCacheStrategyOf(DiskCacheStrategy.RESOURCE)) // decoded resource 를 Disk Cache 저장소에 저장해둔다
                    .into((ImageView) findViewById(R.id.splash_image));
```

## SignIn Activity

### _구조 및 기능_

![signin](https://user-images.githubusercontent.com/31605792/35350274-c9d2f75a-0180-11e8-930f-514c96be12a7.png)
- 일반 로그인과 google 로그인을 하는 로직이 담긴 Activity

### _issue_

- 자동로그인을 위한 token 값 저장
- 로그인에 필요한 로직
- google logIn

### _How to solve?_

> __Preference와 Token값을 사용해 자동로그인 처리__

- 아래의 로직은 preference를 사용해 값을 저장하는 로직으로써, 확장성을 고려해서 PreferenceUtil이라는 클래스를 따로 만들어서 관리
 
```Java
 PreferenceUtil.setStringValue(this, ConstantUtil.AUTHORIZATION_FLAG, response.getToken());
```

> __정규식과 rxbinding을 사용해 로그인에 필요한 조건 처리__

- util성 클래스를 따로 만들어, SignIn Activity에서 이를 활용해서 처리

```Java
public class ValidationUtil {
    private static final Pattern EMAIL_REGEX = Pattern.compile("^[_0-9a-zA-Z-]+@[0-9a-zA-Z-]+(.[_0-9a-zA-Z-]+)*$");
    private static final Pattern PW_REGEX = Pattern.compile("^[a-zA-Z0-9!@.#$%^&*?_~]{8,16}$"); // 8 - 16 자리

    public static boolean isEmpty(String any) {
        if(any == null || "".equals(any))
            return true;
        else
            return false;
    }

    public static boolean isValidEmailAddress(String email_address) {
        return EMAIL_REGEX.matcher(email_address).matches();
    }

    public static boolean isValidPassword(String password) {
        return PW_REGEX.matcher(password).matches();
    }
}
```
- rxbinding을 하면 기존의 textWatcher를 활용한 것보다 코드가 간결하다.

```Java
Observable<CharSequence> o1 = RxTextView.textChanges(editTextEmail);
        Observable<CharSequence> o2 = RxTextView.textChanges(editTextPassword);

        // Observable 1 과 Observable 2 를 결합해서 결과값을 반환
        Observable.combineLatest(o1, o2,
                (e, p) -> ValidationUtil.isValidEmailAddress(e.toString()) &&
                        ValidationUtil.isValidPassword(p.toString()))
        .subscribe(
                (validity) -> {
                    if(validity) {
                        buttonSignIn.setTextColor(Color.BLACK);
                    } else {
                        buttonSignIn.setTextColor(Color.LTGRAY);
                    }
                }
        );
```

## SignUp Activity

### _구조 및 기능_

![signup](https://user-images.githubusercontent.com/31605792/35350272-c9792e46-0180-11e8-9d4a-88c1096c43de.png)
- 회원가입 과정을 통해 서버에 해당 정보를 등록하는 기능을 가진 Activity

### _issue_

- 회원가입 시 요구되는 id, password 조건 
- 서버에 정보를 등록하는 networking

### _How to solve?_

> __정규식으로 회원가입시 요구되는 조건 처리__

- 회원가입 전에 email, password 등의 정보들을 등록하는 과정은 위에서 언급했던 ValidationUtil을 사용해서 처리한다.
- 정의해놓은 조건들을 모두 충족하게 되면, 서버에 등록하는 networking이 진행된다. 

```Java
 findViewById(R.id.button_sign_up).setOnClickListener(
                v -> {
                    // progress bar
                    String email = editTextEmailAddress.getText().toString();
                    String password = editTextPassword.getText().toString();
                    String first_name =  editTextFirstName.getText().toString();
                    String last_name = editTextLastName.getText().toString();
                    if( !ValidationUtil.isValidEmailAddress(email) ) {
                        // email 형식이 틀렸습니다
                    } else if( !ValidationUtil.isValidPassword(password) ) {
                        // password 를 올바르게 입력해주세요 8 - 16 자
                    } else if ( ValidationUtil.isEmpty(first_name) || ValidationUtil.isEmpty(last_name) ) {
                        // 이름을 입력해주세요
                    } else {
                        // request body setting
                        SignUpRequestBody signUpRequestBody = new SignUpRequestBody();
                        signUpRequestBody.setEmail(email);
                        signUpRequestBody.setPassword(password);
                        name = first_name+" "+last_name;
                        signUpRequestBody.setName(first_name+" "+last_name);

                        // retrofit
                        RetrofitHelper.createApi(UserService.class).signUp(signUpRequestBody)
                                .subscribeOn(Schedulers.io())
                                .observeOn(AndroidSchedulers.mainThread())
                                .subscribe(this::handleResponse, this::handleError);
                    }
                }
        );
```


> __retrofit을 활용한 POST방식으로 회원가입 등록 처리__

- 아래는 retrofit2을 사용하기 위한 interface양식이다.
- Body를 POST방식으로 서버에 보내주면, 서버에서 받아서 Body에 담긴 정보들을 User모델에 세팅하는 과정이 실행된다.

```Java
@POST("user/sign-up")
    Observable<UserResponse> signUp(@Body SignUpRequestBody body);
```

## Main Activity

### _구조 및 기능_

- App의 기둥이 되는 Activity로써, 6개의 fragment와 Bottom Navigation 등으로 구성되어 있다.
- __검색기능__ 및 __frament들을 관리__하는 기능이 있다.
- 로그인 및 로그아웃 등의 상황에 따른 로직을 관리하여 각 fragment에 필요한 데이터를 전달한다. 

### _issue_

- NavigationBar item에 전환에 따른 fragment 세팅
- googleGcm 세팅
- 로그인, 로그아웃 시에 따른 상황별 로직

### _How to solve?_

> __getSupportFragmentManager와 replace를 사용해서 처리__

- NavigationBar 아이템에 따른 fragment의 add 및 replace를 컨트롤하는 로직은 아래와 같다.
- (참고) 총 6 개의 프래그먼트가 있고, 회원(비회원)이라는 조건에 따라 프로필을 담당하는 Mefragment(OffLineFragment)는 둘 중 하나만 NavigationBar에 세팅이 된다.
- (참고2) 총 여섯개의 case가 있으나, 로직은 아래와 같기때문에 생략한다. 

- TODO 체크필요
```Java

  bottomNavigation.setOnTabSelectedListener((int position, boolean wasSelected) -> {
            switch (position) {
                case 0:
                    if (wasSelected) {
                        Log.d("JUWONLEE", "0 was selected");
                        return false;
                    } else {
                        if(homeFragment == null) {
                            Log.d("JUWONLEE", "0 null");
                            homeFragment = new HomeFragment();

                            fragments.add(homeFragment);

                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .add(R.id.fragment_container, homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        } else if(fragments.contains(homeFragment)) {
                            Log.d("JUWONLEE", "0 contains");
                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .hide(fragments.get(stack))
                                    .show(homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        } else {
                            Log.d("JUWONLEE", "0 else");
                            getSupportFragmentManager()
                                    .beginTransaction()
                                    .hide(fragments.get(stack))
                                    .show(homeFragment)
                                    .commit();

                            stack = fragments.indexOf(homeFragment);
                        }
                        return true;
                    }
```

> __google GCM guide 참고__

![gcm](https://user-images.githubusercontent.com/31605792/35338510-7a769d02-0161-11e8-842b-db976779ef3d.png)

- 댓글을 달거나, 좋아요 등의 이벤트가 발생했을 때을 위한 notification관련 로직이다.


```Java
   // Broad Cast Receiver
    private void registerReceiver() {
        LocalBroadcastManager localBroadcastManager = LocalBroadcastManager.getInstance(this);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(ConstantUtil.RECEIVED);
        localBroadcastManager.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                if (intent.getAction().equals(ConstantUtil.RECEIVED)) {
                    // notification 받으면 bottom navigation view 에 noti 달기
                    bottomNavigation.setNotification(" ", 1);
                }
            }
        }, intentFilter);
    }
```
```Java
 private void startRegisterService() {
        if(stateUtil.getState()) {
            Intent intent = new Intent(this, RegistrationIntentService.class);
            intent.putExtra("USER_ID", stateUtil.getUserInstance().get_id());
            startService(intent);
        }
    }
```
- 위의 기능을 사용하기위해 google에서 제공하는 상세 로직들은 project내 GCM package에 있으며, 해당 package 관련 핵심 내용은 아래와 같다.

> (참고) GCM 관련 핵심요약
- MyGcmListenerService은 서버에서 푸시를 보내면 수신하여 처리하는 부분
- MyInstanceIDListenerService 각 디바이스마다 토큰이라고 하는 고유 값이 생성되는데 이러한 토큰의 신규 발급, 순환, 업데이트가 발생될때 처리하는 부분
- RegistrationIntentService 토큰을 생성하는 부분
- 참고 블로그 [구글gcm](https://developers.google.com/cloud-messaging/)

> (참고) localBroadCast 장점
- 현재 프로세스 안에서만 유효한 Broadcast
- Global Broadcast 에 비해 훨씬 시스템 부하가 적다. 
- 액티비티 내부의 객체간에 상호 의존성을 낮추어 깔끔한 프로그램 구조를 만들 수 있다는 점이 장점이다.
- 출처: [localbraodCast](https://m.blog.naver.com/PostView.nhn?blogId=rosaria1113&logNo=205338870&proxyReferer=https%3A%2F%2Fwww.google.co.kr%2F)


## HomeFragment(in MainActivity)

### _구조 및 기능_

![home](https://user-images.githubusercontent.com/31605792/35291153-c4844b86-00af-11e8-9dfa-f0cf2a9af9a3.png)
- 위와 같이 아이템들이 바둑판 형식으로 구성되어 있는 구조

### _issue_

- 바둑판 형식으로 구성되어 있는 레이아웃구조
- 동적으로 추가될 수 있는 아이템들을 고려한 로직 

### _How to solve?_

> __중첩된 recyclerView로 처리__

- 수직 방향으로 설정한 recyclerView 안에 아이템으로써 수평방향으로 설정한 recyclerView를 세팅
- 편의상 필요한 부분의 로직만 가지고 옴.
- 아래와 같이 holder에 리싸이클러뷰를 만들어 주고, onBindViewHolder에 수평방향의 recyclerView를 만들어줌으로써 처리했다.
- outer recyclerView안에 아이템으로써의 inner recycler뷰를 map<String, List<T>>가 같은 형태로 으로 제어했다.
- 그 이유는 추후에 inner recycler뷰가 동적으로 추가될 경우가 있고, 그 안에 데이터를 넣기위해서는 map형태로 키값을 찾아 그 안에 데이터를 넣게 해주기 위함이다.   
- inner RecyclerView는 일반적인 방법과 같으므로 생략한다. 

```Java
public class HomeRecyclerViewAdapter extends RecyclerView.Adapter<HomeRecyclerViewAdapter.Holder> {
    List<Map<String, List<Class>>> classes;

    @Override
    public void onBindViewHolder(Holder holder, int position) {
        Map<String, List<Class>> map = classes.get(position);

        for (String title : map.keySet()) {
            holder.textViewTitle.setText(title);
            holder.generalRecyclerView.setLayoutManager(new LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false));
            holder.generalRecyclerView.setAdapter(new GeneralRecyclerViewAdapter(context, map.get(title)));
        }
    }


    class Holder extends RecyclerView.ViewHolder {

        TextView textViewTitle, buttonSeeAll;
        RecyclerView generalRecyclerView;

        public Holder(View view) {
            super(view);
            textViewTitle = view.findViewById(R.id.text_view_title);
            generalRecyclerView = view.findViewById(R.id.general_recycler_view);
            buttonSeeAll = view.findViewById(R.id.button_see_all);
            
        }
    }

}

```

## classActivity

### _구조 및 기능_

![class](https://user-images.githubusercontent.com/31605792/35291171-d9447ca8-00af-11e8-90c7-90ad6a3c9f34.png)

- 수업컨텐츠를 클릭했을때 보여지는 Activity이다.
- skillShare에서 제공하는 핵심 컨텐츠
- 동영상을 시청할 수 있는 기능이 있다.
- 탭레이아웃 + 프래그먼트 구조를 지니고 있다.

### _issue_

- Activity와 Fragment간 통신
- exoLibrary를 활용한 동영상 시청하기 기능

### _How to solve?_

> __bundle을 통해 통신__

- tabLayout과 ViewPager 그리고 3개의 프래그먼트를 사용해서 레이아웃을 구성했다.
- bundle을 통해 해당 class의 id값만 전달한다. 
- 각 fragment에서는 해당 id값으로 필요한 데이터를 서버에서 불러오는 로직을 구현했다. 

```Java
 private void setTabLayout() {
        tabLayout.addTab(tabLayout.newTab().setText("Lessons"));
        tabLayout.addTab(tabLayout.newTab().setText("About"));
        tabLayout.addTab(tabLayout.newTab().setText("Discussion"));

        tabLayout.addOnTabSelectedListener(new TabLayout.ViewPagerOnTabSelectedListener(tabPager));
    }

    private void setTabPager() {
        List<Fragment> fragmentList = new ArrayList<>();

        Bundle bundle = new Bundle();
        bundle.putString(ConstantUtil.ID_FLAG, classId); // 선택된 클래스의 id값만 bundle에 담았고, 각 프래그먼트는 그 id값으로 서버와 통신할 것이다.

        lessonsfragment = new LessonsFragment();
        lessonsfragment.setArguments(bundle); // bundle값 전달
        aboutfragment = new AboutFragment();
        aboutfragment.setArguments(bundle);// bundle값 전달
        discussionsfragment = new DiscussionsFragment();
        discussionsfragment.setArguments(bundle);// bundle값 전달

        fragmentList.add(lessonsfragment);
        fragmentList.add(aboutfragment);
        fragmentList.add(discussionsfragment);

        tabPager.setAdapter(new FragmentAdapter(getSupportFragmentManager(), fragmentList));
        tabPager.addOnPageChangeListener(new TabLayout.TabLayoutOnPageChangeListener(tabLayout));
    }
```


> __ExoPlayer library를 사용하여 동영상 처리__ 

![Exoplayer](https://user-images.githubusercontent.com/31605792/35338505-79c72cd2-0161-11e8-8900-3f88c5fa0483.png)

- 세팅과정
    1. ExoPlayer를 프로젝트에 종속성으로 추가한다.
    2. SimpleExoPlayer인스턴스를 만든다 .
    3. 플레이어를 뷰에 연결한다 (비디오 출력 및 사용자 입력 용).
    4. MediaSource재생할 플레이어를 준비한다.
    5. 완료되면 플레이어를 넣는다.

- 커스터마이징
    - Renderer- Renderer라이브러리가 제공하는 기본 구현에서 지원되지 않는 미디어 유형을 처리 하는 사용자 정의를 구현할 수 있습니다.(media를 decode하고 render해주는 역할을 한다.)
    - TrackSelector- 커스텀을 구현 TrackSelector하면 앱 개발자가 a MediaSource로 노출 된 트랙을 사용 가능한 각각의 트랙 으로 선택 하는 방식을 변경할 수 있습니다.(선택된 트랙을 rendering and playback 해주는 역할을 한다.)
    - LoadControl- 사용자 정의를 구현 LoadControl하면 앱 개발자가 플레이어의 버퍼링 변경할 수 있다. 
    - Extractor- 라이브러리에서 현재 지원되지 않는 컨테이너 형식을 지원해야하는 경우 사용자 정의 Extractor클래스를 구현하는 것을 고려한 다음 ExtractorMediaSource해당 유형의 미디어를 재생 하는 데 함께 사용할 수 있습니다.(특정 MediaSource를 버퍼하는 방식을 컨트롤하는 역할을 한다.)
    - MediaSource- 사용자 정의 MediaSource클래스를 구현하는 것은 사용자 정의 방법으로 렌더러에 피드 할 미디어 샘플을 얻고 싶거나 사용자 정의 MediaSource합성 동작 을 구현하려는 경우에 적합 할 수 있습니다. (어떤 media가 재생될지 어떻게 load 될지 컨트롤 해주는 역할)
    - DataSource- ExoPlayer의 업스트림 패키지에는 DataSource다양한 사용 사례에 대한 여러 가지 구현이 이미 포함되어 있습니다 . DataSource사용자 지정 프로토콜을 통해, 사용자 지정 HTTP 스택을 사용하여 또는 사용자 지정 영구 캐시에서와 같이 다른 방식으로 데이터를로드하기 위해 자체 클래스 를 구현할 수 있습니다 .

- [exoplayerGuide](http://google.github.io/ExoPlayer/guide.html)

- [전체 코드]()

```java
private void initiatePlayer() {
        userAgent = Util.getUserAgent(this, "Skill Share");

        // view
        simpleExoPlayerView = new SimpleExoPlayerView(this);
        simpleExoPlayerView = findViewById(R.id.simple_exo_player_view);
        simpleExoPlayerView.requestFocus(); // ( ? )
        simpleExoPlayerView.setUseArtwork(true);
        simpleExoPlayerView.setUseController(true); //Set media controller
        simpleExoPlayerView.setControllerHideOnTouch(false);
        simpleExoPlayerView.showController();

        // renders [ 4, 5 ]
        DefaultRenderersFactory renderersFactory = new DefaultRenderersFactory(this, null);

        // track selector [ 5 ]
        TrackSelection.Factory adaptiveTrackSelectionFactory =
                new AdaptiveTrackSelection.Factory(BANDWIDTH_METER);
        trackSelector = new DefaultTrackSelector(adaptiveTrackSelectionFactory);

        player = ExoPlayerFactory.newSimpleInstance(renderersFactory, trackSelector);
        player.addListener(new PlayerEventListener());

        // binding
        simpleExoPlayerView.setPlayer(player);
        player.setPlayWhenReady(false);

        // dash
        // data source factory
        mediaDataSourceFactory = buildDataSourceFactory(true);

        // [ 1, 2, 3 ( ? ) ]
        mediaSource = buildMediaSource(Uri.parse(URL));

        if(resumePosition > 0)
            player.seekTo(resumePosition);
        player.prepare(mediaSource);
    }
```
- 아래는 비디오가 실행되면서 생길 수 있는 다양한 상황에서 원활한 재생을 위해 custome한 method이다.

```Java
 private void saveResumePosition() {// 현재 재생된 시간까지 저장
        
        resumePosition = player.getContentPosition();
        resumePosition = (resumePosition > 0) ? resumePosition : 0;
    }


    private void releasePlayer() { // player 객체가 있으면, 저장된 시간부터 시작
        if(player != null) {
            saveResumePosition();
            player.release();
            player = null;
            trackSelector = null;
        }
    }

    @Override                           
    protected void onPause() {  // 생명주기를 활용해 잠깐 정지시에 멈춤
        player.setPlayWhenReady(false);
        super.onPause();
    }

    @Override
    protected void onDestroy() {// 앱이 꺼졌을때도 상태저장
        releasePlayer();
        super.onDestroy();
    }
```


## LessonFragment(in ClassActivity)

### _구조 및 기능_

![lesson](https://user-images.githubusercontent.com/31605792/35291171-d9447ca8-00af-11e8-90c7-90ad6a3c9f34.png)
- content제작자(tutor)에 대한 정보를 나타내는 Activity

### _issue_

- Follow, Unfollow 토글버튼 클릭 이벤트 처리

### _How to solve?_

> __아래 코드 참고__

```Java

if (StateUtil.getInstance().getState()) {   // 로그인을 하고 들어온 경우
            List<Following> followings = StateUtil.getInstance().getUserInstance().getFollowing();
            for (Following following : followings) {            // user가 가지고 있는 following 모델안에 데이터를 싹 돌면서,
                if (following.getUserId().equals(tutorId)) {   // userId와 tutorId가 같은게 있다면, 이미 follwing을 했다는 의미이므로
                    buttonFollow.setChecked(true);              // 체크가 된 상태로 뷰가 생성된다.
                    buttonFollow.setTextColor(getResources().getColor(R.color.white));
                    break;
                }
            }

            buttonFollow.setOnCheckedChangeListener((buttonView, isChecked) -> {
                if (isChecked) {    //toggle버튼이 체크가 되면, followers는 한명 증가
                    buttonView.setTextColor(getResources().getColor(R.color.white));
                    followers++;
                    toggleFollow();
                } else { // 다시 toggle버튼이 체크가 되면 followers 원상 복귀
                    buttonView.setTextColor(getResources().getColor(R.color.IcActive));
                    followers--;
                    toggleFollow();
                }

                textViewFollowersCount.setText(followers + " Followers");
            });

        } else { // 로그인이 안되어있을때는 popup으로 로그인창이 뜬다.
            buttonFollow.setOnCheckedChangeListener((buttonView, isChecked) ->
                    {
                        buttonView.setChecked(false);
                        DialogUtil.showSignDialog(context);
                    }
            );
        }
```


## AboutFragment(in ClassActivity)

### _구조 및 기능_

![about](https://user-images.githubusercontent.com/31605792/35291194-eeb2c04a-00af-11e8-9506-6d04b8fae529.png)
- 해당 컨텐츠가 가지고 있는 프로젝트, 리뷰, 구독자, 연관된 클래스를 정보를 나타내는 Activity

### _issue_

- seeAll(더보기)관련 처리

### _How to solve?_

> __constantUtil을 활용하여 '더보기' 처리__

- 앱내에 많은 더보기 기능을 처리하기 위해 constantUtil을 활용했다.
- 더보기 클릭 이벤트가 일어날때 해당 키값으로 constantUtil에 정의 해놓은 값을 사용한다.
- 예를 들어, 더보기 클릭 이벤트에 다음과 같이 정의해놓고, 
```java
 textViewReviewSeeAll.setOnClickListener(view -> {
                Intent intent = new Intent(context, SeeAllActivity.class);
                intent.putExtra(ConstantUtil.SEE_ALL_FLAG, ConstantUtil.REVIEW_ITEM);
                intent.putExtra(ConstantUtil.ID_FLAG, classId);
                intent.putParcelableArrayListExtra(ConstantUtil.REVIEWS_FLAG, (ArrayList<? extends Parcelable>) reviews);
                context.startActivity(intent);
            });
```
- SeeAll Activity
```Java
// ...중략

    intent = getIntent(); 
    //이와 같이 seeAll 클릭이벤트시 전달한 intent를 한 액티비티에서 일괄적으로 전달받는다. 
    type = intent.getStringExtra(ConstantUtil.SEE_ALL_FLAG); 
    // 전달시에 key값은 ConstantUtil.SEE_ALL_FLAG으로 통일하고, value는 이벤트가 일어나는 곳의 특징을 할 수 있는 naming을 해서 전달해주기 때문에 아래와 같은 로직으로 처리 할 수 있다.

//...중략


 switch (type) { 
     // type에는 위에 주석으로 남겨놓은 것 처럼, 해당 이벤트가 일어난 곳의 특징을 알 수 있는 값이 담겨 있을 것이므로, case문을 자동으로 해당 값에 맞는 곳에 들어가 로직이 실행된다.
            case ConstantUtil.CLASS_ITEM:

                break;
            case ConstantUtil.REVIEW_ITEM:
                // 예를 들어, 위와 같은 경우 ConstantUtil.REVIEW_ITEM를 값을 보냈기에, type은 ConstantUtil.REVIEW_ITEM으로 정의가 되어 있을 것이므로, 이곳에 작성한 로직대로 동작을 할 것이다.
                ...생략

                // Intent를 다음과 같이 ConstantUtil에 명시해놓은 키값으로 받아서 사용한다.
                List<Review> reviews = intent.getParcelableArrayListExtra(ConstantUtil.REVIEWS_FLAG);

                int size = reviews.size();
                int likeCount = 0;
                for(Review review : reviews) {
                    if("like".equals(review.getLikeOrNot()))
                        likeCount++;
                }

                //...중략

                recyclerView.setAdapter(new ReviewSeeAllRecyclerViewAdapter(this, reviews));
                
                break;  // break은 항상 잊지 않는다.
```


## discussionFragment(in ClassActivity)

### _구조 및 기능_


![discussion](https://user-images.githubusercontent.com/31605792/35350389-21a1bcc8-0181-11e8-91f5-36c5e79b86ba.png)
- 로그인이 되어 있는 사용자들이 자신들의 의견을 기재하는 Activity
- 의견과 의견에 대한 [댓글 및 좋아요]가 가능하다.(댓글, 대댓글의 구조)


### _issue_

- [의견]과 [의견에 대한 댓글] 및 [좋아요] 기능 구현
- [의견이 있는 프래그먼트]와 [해당 의견에 대한 모든 댓글이 있는 액티비티]간에 원할한 데이터 전달

### _How to solve?_

> __의견을 입력하는 칸에 텍스트 입력 후 button이벤트가 발생할떄 의견(discussion) model을 만들어 서버에 전송__

- 아래와 같은 method를 버튼 이벤트 일어날때 세팅시킨다.

```Java
public void sendDiscussion() {
        String input = editText.getText().toString(); // edittext view에 입력하는 값을 가지고 있는 input 변수 선언
        if( !ValidationUtil.isEmpty(input) ) { /
            editText.setText("");                      // 초기화
            User user = StateUtil.getInstance().getUserInstance(); // 싱글톤으로 가지고 온 user 객체
            Discussion리discussion = new Discussion(     // 정의 해놓은 Discussion 모델에 필요한 정보를 기입.
                    user.getName(),
                    user.getImageUrl(),
                    input,                              // 사용자가 입력한 내용
                    System.currentTimeMillis()+"",
                    "0",
                    user.get_id(),
                    user.getRegistrationId(),
                    new ArrayList<>(),
                    new ArrayList<>()
            );

            RetrofitHelper.createApi(ClassService.class)
                    .sendDiscussion(discussion, classId)
                    .subscribeOn(Schedulers.io())
                    .observeOn(AndroidSchedulers.mainThread())
                    .subscribe(this::handleResponse, this::handleError);

            // 사용자의 정보 및 해당 클래스의 id를 서버로 POST방식으로 보낸다.
           
           List<Discussion> discussions;

private void handleResponse(List<Discussion> discussions) { 
        Collections.reverse(discussions); // 서버에서 데이터를 받아올떄, Collections.reverse()를 함으로써, 가장 최신의 의견을 가장 위로 세팅시킨다.
        this.discussions = discussions;

        // TODO list reverse or get data by sort
        
        if(discussions == null || discussions.size() == 0) {
            textViewDiscussion.setVisibility(View.GONE);
        } else {
            if(textViewDiscussion.getVisibility() == View.VISIBLE) {// update 시에는 새롭게 갱신해준다. >>> recycler view 위치 초기화
                adapter.updateData(discussions);
            } else {
                adapter.initiateData(discussions);
                textViewDiscussion.setVisibility(View.VISIBLE);
            }

            textViewDiscussion.setText(discussions.size() + " Discussions");
        }
        // TODO hide progress bar
    }
        }
    }
```

> __의견에 대한 [댓글]과 [좋아요]는  DiscussionsAdapter에서 처리, 의견에 대한 모든 [댓글]은 DiscussionSeeAllRecyclerViewAdapter 처리__

- 리싸이클러뷰 아이템을 의견과 댓글이 합쳐진 형태로 만들고, 조건에 따라 visible, invisible로 처리
- 의견에 대한 좋아요는 DiscussionAdapter onBindViewHolder에서 로직처리
- 의견에 대한 댓글은 constantUtil 값을 사용, SeeAllActivity으로 인텐트에 정보 넘겨서 로직 처리

- [좋아요] 처리 로직
```Java
// 좋아요 처리

if(StateUtil.getInstance().getState()) {
                if(discussion.getLikeUsersIds() != null && discussion.getLikeUsersIds().size() != 0) {
                    String userId = StateUtil.getInstance().getUserInstance().get_id();
                    if(discussion.getLikeUsersIds().contains(userId)) // 사용자(나)가 좋아요를 이미 누른 경우, 좋아요 버튼은 클릭이 되어 있는 상태로 세팅
                        holder.imageButtonLike.setChecked(true);
                }

                // 이하 좋아요를 눌렀을때, 취소했을 때 처리
                User user = StateUtil.getInstance().getUserInstance();
                holder.imageButtonLike.setOnCheckedChangeListener(
                        (buttonView, isChecked) -> {
                            if(isChecked) {
                                RetrofitHelper.createApi(ClassService.class)
                                        .like(new LikeRequestBody(holder.discussionId, user.get_id(), user.getName(), holder.resId))
                                        .subscribeOn(Schedulers.io())
                                        .observeOn(AndroidSchedulers.mainThread())
                                        .subscribe(
                                                (Response response) -> {
                                                    String likeCount = response.getResult();
                                                    discussion.setLikeCount(likeCount);
                                                    holder.textViewLikeCount.setText(likeCount);
                                                }, (Throwable error) -> {
                                                    Log.d("JUWON LEE", "error : " + error.getMessage());
                                                }
                                        );
                            } else {
                                RetrofitHelper.createApi(ClassService.class)
                                        .unLike(new LikeRequestBody(holder.discussionId, user.get_id(), user.getName(), holder.resId))
                                        .subscribeOn(Schedulers.io())
                                        .observeOn(AndroidSchedulers.mainThread())
                                        .subscribe(
                                                (Response response) -> {
                                                    String likeCount = response.getResult();
                                                    discussion.setLikeCount(likeCount);
                                                    holder.textViewLikeCount.setText(likeCount);
                                                }, (Throwable error) -> {
                                                    Log.d("JUWON LEE", "error : " + error.getMessage());
                                                }
                                        );
                            }
                        }
                );
            } 
```

- [의견에 대한 댓글] 처리로직

```Java
 public void setReply(List<Reply> replies) {

            this.replies = replies;

            if(replies != null && replies.size() != 0) { 
                textViewReplyProfile.setVisibility(View.VISIBLE);
                textViewTimeReply.setVisibility(View.VISIBLE);
                imageViewReplyProfile.setVisibility(View.VISIBLE);
                expandableTextViewReply.setVisibility(View.VISIBLE);

                int size = replies.size();
                if( size >= 2 ) {
                    textViewReplies.setText("See all " + replies.size() + " replies");
                    textViewReplies.setVisibility(View.VISIBLE);
                    textViewReplies.setOnClickListener(this);
                }

                Reply reply = replies.get(replies.size()-1);
                textViewReplyProfile.setText(reply.getName());
                textViewTimeReply.setText( TimeUtil.calculateTime(reply.getTime()) );

                if(reply.getImageUrl() != null)
                    Glide.with(context).load(reply.getImageUrl())
                            .apply(RequestOptions.circleCropTransform())
                            .into(imageViewReplyProfile);

                expandableTextViewReply.setTrimLength(4);
                expandableTextViewReply.setText(reply.getContent(), TextView.BufferType.NORMAL);
            } else {
                textViewReplyProfile.setVisibility(View.GONE);
                textViewTimeReply.setVisibility(View.GONE);
                imageViewReplyProfile.setVisibility(View.GONE);
                expandableTextViewReply.setVisibility(View.GONE);
                textViewReplies.setVisibility(View.GONE);
            }
        }
```

> __Fragment와 Activity간 원활한 통신은 eventBus library의 작동방식을 참고로 해서 Cusomize한 방식으로 처리__

- EventBus library?

![EventBus](http://cfile4.uf.tistory.com/image/2271424F5591DD5D0B5077)

- 사용방법

```java

1. 이벤트 리스너를 등록해준다(Subscriber)
2. 이벤트가 발생했을때 수행할 작업을 만들어 둔다(Subscriber)
3. 이벤트가 발생하는 위치에서 이벤트가 발생했다고 알린다(Publisher)
4. 해당 이벤트에서 원하는 변수,리스트 등을 받아서 작업을 처리한다(Subscriber)
5. 이벤트 리스너를 등록해제해준다(Subscriber)

출처: http://gun0912.tistory.com/4 [박상권의 삽질블로그]

```

- 아래에 코드는 eventBus를 커스터마이징 해서 적용했다.
- reflection이라는 자바 API 사용(구체적인 클래스 타입을 알지 못해도, 그 클래스의 메소드, 타입, 변수들을 접근 할 수 있도록 해주는 자바 API)

```Java

public class Bus { // 이 클래스가 필요한 프래그먼트 및 액티비티가 더 있기때문에 따로 클래스를 만들어 모듈화

    Map<Class<?>, Subscription> subscriptionByType; // 맵 형태로 클래스별로 해당 구독자를 구별할 수 있다. 

    static Bus instance; // static으로 선언함으로써 어느 곳에서나 사용가능

    public static Bus getInstance() { // 싱글톤으로 만들어주면서, 객체의 유일성 보장
        if(instance == null) {
            instance = new Bus();
        }
        return instance;
    }

    private Bus() { // 여러 클래스에서 사용이 될 것이므로, map으로 초기화 해준다.
        subscriptionByType = new HashMap<>();
    }

    public void register(Object subscriber) { // 이 구독이라고 명명한 메소드는 object타입을 인자를 받음으로써, 해당 클래스를 객체를 받아오기 위한 것이다.
        Class<?> subscriberClass = subscriber.getClass(); // 해당 클래스를 얻어온다는 의미
        SubscriberMethod subscriberMethod = getMethod(subscriberClass); // 해당 클래스의 메소드를 가져와서 세팅, getMethod 아래에 정의
        subscribe(subscriber, subscriberMethod); // subscribe 아래 정의되어 있음.
    }

    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        if(subscriptionByType != null) {
            subscriptionByType.put(subscriberMethod.type, new Subscription(subscriber, subscriberMethod)); 
            // map으로 정의 해놓은 subscriptionByType에 미리지정한 subscriberMethod 클래스와 Subscription 클래스에 형태에 맞게 인자를 넣는다. 
        }
    }

    public void post(Map<Integer, List<Reply>> data) {
        invokeSubscriber(subscriptionByType.get(java.util.Map.class), data);
    }

    private void invokeSubscriber(Subscription subscription, Map<Integer, List<Reply>> data) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, data); // invoke() : Method 클래스의 메서드, 동적으로 호출된 메서드의 결과(Object) 리턴
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    //...중략

    private SubscriberMethod getMethod(Class<?> subscriberClass) { 
        for(Method method : subscriberClass.getMethods())   // getMethods()는 클래스API에 정의 되어 있다. Naming 그대로, 클래스의 메소드을 배열형태로 가지고 온다 
        {                                                   // 해당 클래스의 메소드들은 여러개인 경우가 대부분이기에 반복문을 돌면서,
             Subscribe annotation = method.getAnnotation(Subscribe.class);  // 반복문을 돌면서 해당 클래스에 지정한 어노테이션을 구한다. 슈퍼 클래스의 어노테이션도 포함한다.
            if(annotation != null) { // annotation null check
                Class<?>[] parameters = method.getParameterTypes();         // 해당 메소드의 파라미터 타입 알아내는 과정
                if(parameters.length == 1) {
                    return new SubscriberMethod(method, parameters[0]); 
                }
            }
        }

        return null;
    }
}
```
- Subscribe Interface

```Java
@Documented // doc 에 '어노테이션' 정보 저장
@Retention(RetentionPolicy.RUNTIME) // RUNTIME : '어노테이션' 이 class 파일에 저장되고 읽혀진다.

// @Retention(RetentionPolicy.CLASS) // 컴파일러가 클래스를 참조할 때까지 유효하다.
// @Retention(RetentionPolicy.SOURCE) // 어노테이션 정보는 컴파일 이후 없어진다.

@Target({ElementType.METHOD}) // 해당 '어노테이션' 을 붙일 element 타입 설정
public @interface Subscribe {
//    boolean isSubscribe() default true;
}
```
- Subscription, SubscriberMethod 클래스는 communication_util package에 있으므로 생략한다.
- 이렇게 커스터마이징을 한 클래스를 사용하는 코드는 아래와 같다. 
- 이벤트를 등록하는 부분의 코드

```Java

 Bus.getInstance().register(this);

```
- 등록한 이벤트를 불러오는 코드

```Java

Bus.getInstance().post(data);

```


## GroupFragment(in MainActivity)

### _구조 및 기능_

![group](https://user-images.githubusercontent.com/31605792/35321978-faa8238c-012b-11e8-92fc-b28c45541390.png)

### _issue_

- 화면 구성 및 통신
- groupActivity와의 통신(group Activity에서 [join group]를 누르면 groupFragment의 [my Groups]이라는 카테고리 밑에 해당 Group이 추가되는 것)
- 즉, [my Groups]라는 하나의 View안에는 recyclerView가 있는데 그 안에 item이 group Activity에서 버튼 이벤트에 의해 생성되거나 사라지는 것

### _How to solve?_

> __화면구성은 ScrollView안에 3개의 RecyclerView를 만들어 구성했고, GroupActivity와 통신을 해야하므로 Custom한 Bus를 사용해 등록해서 처리했다.__

- original application 분석결과, homeFragment와 달리 GroupFragment는 RecyclerView가 사용자의 행위에 따라 추가생성 되지 않기 때문이다.
- 상기와 같은 이유로, 각 카테고리마다 RecyclerView 세팅을 하는 과정이 있다. 

```JAVA
@Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_group, container, false);

        context = getActivity();
        // 카테고리별 RecyclerView setting : my Groups
        RecyclerView myGroupsRecyclerView = view.findViewById(R.id.my_groups_recycler_view);
        myGroupsRecyclerView.setLayoutManager(new LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false));
        mAdapter = new GroupRecyclerViewAdapter(context);
        myGroupsRecyclerView.setAdapter(mAdapter);
        // 카테고리별 RecyclerView setting : feature Groups
        RecyclerView featuredGroupsRecyclerView = view.findViewById(R.id.featured_groups_recycler_view);
        featuredGroupsRecyclerView.setLayoutManager(new LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false));
        fAdapter = new GroupRecyclerViewAdapter(context);
        featuredGroupsRecyclerView.setAdapter(fAdapter);
        // 카테고리별 RecyclerView setting : recently Active Groups
        RecyclerView recentlyActiveGroupsRecyclerView = view.findViewById(R.id.recently_active_groups_recycler_view);
        recentlyActiveGroupsRecyclerView.setLayoutManager(new LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false));
        rAdapter = new GroupRecyclerViewAdapter(context);
        recentlyActiveGroupsRecyclerView.setAdapter(rAdapter);

        // RecyclerView마다 다른 id가 있는 고유한 View이지만 그 안에 들어가는 데이터 타입이나 형식 등은 같기 때문에 Adapter는 하나로 통일했다.

        RetrofitHelper.createApi(HomeService.class)
                .getGroups()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(this::handleResponse, this::handleError);
        // RecyclerView가 세팅이 되어 있으므로, 서버에서 데이터를 넣어만 주면 된다.

        Bus.getInstance().register(this);
        // GroupActivity와 통신을 위해서 등록하는 과정이다.

        return view;
    }

    private void handleResponse(Map<String, List<Group>> groups) {
          
        StateUtil state = StateUtil.getInstance();

        if(state.getState()) {
            if(state.getUserInstance().getGroups() != null) {
                mAdapter.update(state.getUserInstance().getGroups());
                // my Groups는 사용자(나)가 선택한 그룹들이 있어야 하므로, 사용자 체크(싱글톤)를 해준 후 User model 이 가지고 있는 Groups을 세팅해준다.
            }
        }

        List<Group> featuredGroups = groups.get("Featured Groups");
        fAdapter.update(featuredGroups);

        List<Group> recentlyActiveGroups = groups.get("Recently Active Groups");
        rAdapter.update(recentlyActiveGroups);

        // 상기 2개의 Groups은 Map<String, List<Group>> groups를 인자로 받은 것에 따라, groups.get(object Key)에 따라 서버에서 보내준 String값과 맞으면 list가 들어온다.
     
    }
```
-   어댑터.update(/* list */)가 호출될 때에는 GroupRecyclerViewAdapter 안에 만들어 놓은 update method 아래 다음과 같이 notifyDataSetChanged를 호출해 변동사항을 알려준다. 

```JAVA

    public void update(List<Group> groups) {
        this.groups = groups;
        notifyDataSetChanged();
    }

```
## GroupActivity(in GroupFragment)

### _구조 및 기능_

![groupactivity](https://user-images.githubusercontent.com/31605792/35341393-e93188d6-0168-11e8-86e5-43090a7895f6.png)
- Group member들과 댓글을 달면서 Communication을 할 수 있는 Activity이다.
- GroupFragment에서 item클릭시 이동화면이다.

### _issue_

- 로그인, 로그아웃 시, 그룹멤버일때와 아닐때에 따라 다른 이벤트 처리
- [joinGroups]라는 버튼을 누르면 message를 보낼 수 있는 창으로 변하면서, GroupFragment my Groups 카테고리 밑에 해당 item이 추가
- recyclerView scroll 시에 데이터를 특정 개수만큼만 받아오는 이슈

### _How to solve?_

> __DialogUtil을 사용해 로그인, 로그아웃 시에 따라 다른 이벤트 처리__

```Java
 if(state.getState()) {
            if(state.getUserInstance().getGroups() != null) {
                if(state.getUserInstance().getGroups().contains(mGroup)) {
                    buttonJoinGroup.setVisibility(View.GONE);
                    layout_discussion.setVisibility(View.VISIBLE);
                } else {
                    buttonJoinGroup.setOnClickListener(v -> { // 그룹멤버가 아니고 
                        if(state.getUserInstance().getNickname() == null || state.getUserInstance().getNickname().equals("")) { // 별명이 없을 때
                            AlertDialog dialog = DialogUtil.showSettingNicknameDialog(this); // DialogUtil에 만들어놓은 AlertDialog 사용한다.
                            dialog.setOnDismissListener(d -> { // 이하 dialog View안에서 처리하는 로직들
                                if(state.getUserInstance().getNickname() != null) {
                                    buttonJoinGroup.setVisibility(View.GONE);
                                    layout_discussion.setVisibility(View.VISIBLE);

                                    RetrofitHelper.createApi(UserService.class)
                                            .joinGroup(mGroup, state.getUserInstance().get_id())
                                            .subscribeOn(Schedulers.io())
                                            .observeOn(Schedulers.io())
                                            .subscribe(
                                                    (Response response) -> {
                                                        if(ConstantUtil.SUCCESS.equals(response.getResult())) {
                                                            state.getUserInstance().getGroups().add(mGroup);
                                                            Bus.getInstance().post(state.getUserInstance().getGroups()); // GroupFragment에서 등록한 것을 불러옴.
                                                        } else {
                                                            // TODO 실패 메시지
                                                            Log.d("JUWONLEE","failed");
                                                        }

                                                    }, (Throwable error) -> {
                                                        Log.e("join group error :  ", error.getMessage());
                                                    }
                                            );
                                } else {
                                    buttonJoinGroup.setVisibility(View.VISIBLE);
                                    layout_discussion.setVisibility(View.GONE);
                                }
                            });
                        }
                    });
                }
            } else { // User가 로그인은 되어 있으나, 가지고 있는 Group이 없을 떄
                buttonJoinGroup.setOnClickListener(v -> {
                    buttonJoinGroup.setVisibility(View.GONE);
                    layout_discussion.setVisibility(View.VISIBLE);

                    RetrofitHelper.createApi(UserService.class)
                            .joinGroup(mGroup, state.getUserInstance().get_id())
                            .subscribeOn(Schedulers.io())
                            .observeOn(AndroidSchedulers.mainThread())
                            .subscribe(
                                    (Response response) -> {
                                        if(ConstantUtil.SUCCESS.equals(response.getResult())) {
                                            state.getUserInstance().setGroups(new ArrayList<>()); // user모델에 리스트 생성해주고
                                            state.getUserInstance().getGroups().add(mGroup); // 그 안에 join 그룹한 것을 추가해주고
                                            Bus.getInstance().post(state.getUserInstance().getGroups()); // 해당 그룹에 대한 정보를 가지고 온다.
                                        }

                                        else {
                                            // TODO 실패 메시지
                                        }

                                    }, (Throwable error) -> {
                                        Log.e("join group error :  ", error.getMessage());
                                    }
                            );
                });
            }
        } else {
            buttonJoinGroup.setOnClickListener(v -> {
                DialogUtil.showSignDialog(this);
            });
        }
```
> __Interface, recyclerView listener, 그리고 handler로 데이터를 원하는 만큼 받아오는 것 처리__

- GroupActivity에 recyclerView Sroll시 처리 로직
```Java
mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                super.onScrolled(recyclerView, dx, dy);
                LinearLayoutManager llManager = (LinearLayoutManager) recyclerView.getLayoutManager();
                if (llManager.findLastCompletelyVisibleItemPosition() == (mAdapter.getItemCount() - 4)) { // 스크롤링 하면서, 마지막 item-4정도에
                    mAdapter.setMore(true); // adapter에 true값을 넣고
                    mAdapter.showLoading(); // 더보여주는 함수를 호출한다.(setmore,showLoading관련 메소드는 adapter에서 처리하므로 밑에 참고한다.)
                }
            }
        });
    // 중략...

    //  GroupCommentAdapter에서 정의한 interface onLoadMore를 GroupActivity에서 implements해서 override한 것이다.
    // 참고 public class GroupActivity extends AppCompatActivity implements GroupCommentAdapter.OnLoadMoreListener, GroupCommentAdapter.InteractionInterface{..생략}
    @Override
    public void onLoadMore() {
        RetrofitHelper.createApi(GroupService.class)
                .getComments(groupId, mAdapter.getItemCount()-1)
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                        (List<Comment> comments) -> {
                            mAdapter.dismissLoading(); // 스크롤링하면서 지나간 데이터는 지우고
                            mAdapter.addItemMore(comments); // 새로운 데이터를 추가한다(데이터가 있다면)
                            if(comments.size() < 20) {
                                mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
                                    @Override
                                    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                                        super.onScrolled(recyclerView, dx, dy);
                                    }
                                });
                            }
                        }, (Throwable error) -> {

                        }
                );
    }
```
- 다음은 Adapter에서 setmore, showLoading 메서드 처리

```Java
public class GroupCommentAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    final int VIEW_ITEM = 1;
    final int VIEW_PROG = 0;

    ArrayList<Comment> comments;

    Context context;
    OnLoadMoreListener onLoadMoreListener;
    InteractionInterface interactionInterface;

    boolean isMoreLoading = true;

    public interface OnLoadMoreListener { // 인터페이스 정의, 
        void onLoadMore();
    }

    public GroupCommentAdapter(Context context) {
        this.context = context;

        if(context instanceof OnLoadMoreListener) {
            this.onLoadMoreListener = (OnLoadMoreListener) context;
        }

        if(context instanceof InteractionInterface) {
            this.interactionInterface = (InteractionInterface) context;
        }

        comments = new ArrayList<>();
    }

    @Override
    public int getItemViewType(int position) {
        return comments.get(position) != null ? VIEW_ITEM : VIEW_PROG;
    }

    @Override

    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == VIEW_ITEM) {  // 위에 정의해놓은 getItemViewType에 따라 inflate하는 것에 차이를 두어 처리
            return new GroupChattingHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.recycler_view_item_comment, parent, false)); 
        } else {
            return new ProgressViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_progress, parent, false)); // progressbar 세팅
        }
    }

    public void showLoading() {
        if (isMoreLoading && comments != null && onLoadMoreListener != null) { // 메소드호출한곳에서 이 조건이 맞으면,
            isMoreLoading = false; // true를 false로 바꿔주고, 

            new Handler().post(() -> { // 핸들러 처리를 한다.
                // progress bar holder 호출
                comments.add(null);
                notifyItemInserted(comments.size() - 1); // 아이템 추가
                // data load
                onLoadMoreListener.onLoadMore(); // 콜백
            });
        }
    }

    public void setMore(boolean isMore) { 
        this.isMoreLoading = isMore;
    }

    public void dismissLoading() { // 지나간 데이터는 지운다
        if (comments != null && comments.size() > 0) {
            comments.remove(comments.size() - 1);
            notifyItemRemoved(comments.size());
        }
    }

    public void addAll(List<Comment> lst) {
        comments.clear();
        comments.addAll(lst);
        notifyDataSetChanged();
    }

    public void addItemMore(List<Comment> lst) { 
        int sizeInit = comments.size();
        comments.addAll(lst);
        notifyItemRangeChanged(sizeInit, comments.size()); // comments.size부터 comments.size개수만큼 즉, 예를들어 20번째 부터, 20번이 추가되므로 20~40 
    }

    public void addItem(Comment comment) {
        comments.add(0, comment);
        notifyItemInserted(0);
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        if (holder instanceof GroupChattingHolder) {
            Comment comment = comments.get(position);

            GroupChattingHolder chatHolder = (GroupChattingHolder) holder;

            chatHolder.userId = comment.getUserId();
            chatHolder.textViewName.setText(comment.getUserName());
            chatHolder.textViewNickname.setText(comment.getUserNickname());
            if(comment.getComment().contains("@")) {
                int index = comment.getComment().indexOf("@");

            }
            chatHolder.expandableTextView.setText(comment.getComment());
            chatHolder.textViewTime.setText(TimeUtil.calculateTime(comment.getTime()));
        }
    }


    @Override
    public int getItemCount() {
        return comments.size();
    }

    public class GroupChattingHolder extends RecyclerView.ViewHolder {

        String userId;
        ImageView imageViewProfile;
        TextView textViewName, textViewNickname, textViewTime;
        ExpandableTextView expandableTextView;
        TextView reply;

        public GroupChattingHolder(View v) {
            super(v);
            imageViewProfile = v.findViewById(R.id.image_view_profile);
            imageViewProfile.setOnClickListener(
                    view -> {
                        Intent intent = new Intent(context, ProfileActivity.class);
                        intent.putExtra(ConstantUtil.USER_ID_FLAG, userId);
                        context.startActivity(intent);
                    }
            );
            textViewName = v.findViewById(R.id.text_view_user_name);
            textViewNickname = v.findViewById(R.id.text_view_user_nickname);
            expandableTextView = v.findViewById(R.id.expandable_text_view);
            expandableTextView.setTrimLength(8);
            textViewTime = v.findViewById(R.id.text_view_time);
            reply = v.findViewById(R.id.text_view_reply);
            reply.setOnClickListener(
                    view -> {
                        interactionInterface.reply(textViewNickname.getText().toString());
                    }
            );
        }
    }

    public class ProgressViewHolder extends RecyclerView.ViewHolder {
        public ProgressBar pBar;

        public ProgressViewHolder(View v) {
            super(v);
            pBar = v.findViewById(R.id.pBar);
        }
    }

    public interface InteractionInterface {
        void reply(String nickname);
    }
}
```
## DiscoverFragment(in MainActivity)

### _구조 및 기능_

![discover](https://user-images.githubusercontent.com/31605792/35350625-d99c5ed2-0181-11e8-82e1-27e0f0dae6d3.png)

### _issue_
### _How to solve?_