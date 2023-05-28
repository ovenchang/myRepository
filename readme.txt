Fallback 路由
有其他路由匹配傳入請求時將執行的路由
Fallback 路由應該始終是你的應用程式註冊的最後一個路由
Route::fallback(function () {});

速率限制

表單方法欺騙
<form action="/example" method="POST">
    //blade
    @method('PUT') 
    @csrf
    //html
    <input type="hidden" name="_method" value="PUT">
    <input type="hidden" name="_token" value="{{ csrf_token() }}">
</form>

當前路由資訊
$route = Route::current(); // Illuminate\Routing\Route
$name = Route::currentRouteName(); // string
$action = Route::currentRouteAction(); // string

跨域資源分享
請求將由OPTIONSconfig/cors.phpOPTIONSHandleCors 中間件默認情況下，它包含在您的全域中間件堆疊中。全域中間件堆疊位於應用程式的 HTTP 內核中 （）。App\Http\Kernel

路由緩存
生產環境用 php artisan route:cache   
清除 php artisan route:clear
===========================================================
中間件
檢查和過濾進入應用程序的 HTTP 請求  是否已通過身份驗證 日誌記錄
身份驗證和 CSRF 保護的中間件。這些中間件都位於該app/Http/Middleware目錄中

定義中間件
php artisan make:middleware EnsureTokenIsValid
新增 app/Http/Middleware/EnsureTokenIsValid

//handle處理
//string $role是中間件額外的參數
public function handle(Request $request, Closure $next, string $role): Response
{
    if ($request->input('token') !== 'my-secret-token') {
        return redirect('home');
    }
     return $next($request);
 }

在應用程序處理request之前執行
public function handle(Request $request, Closure $next, string $role): Response
{
        // 處理
        return $next($request);
}

在應用程序處理request後執行
public function handle(Request $request, Closure $next, string $role): Response
{
        $response = $next($request);
        // 處理
        return $response;
}

中間件可能需要在 HTTP response發送到瀏覽器後做一些工作
在中間件加上
public function terminate(Request $request, Response $response): void
{
       
 }
在控制器加上
public function register(): void
{
    $this->app->singleton(TerminatingMiddleware::class);
}



註冊中間件

全局
希望中間件在對每個 HTTP 請求期間運行
app/Http/Kernel.php類的屬性中列出中間件

為路由分配中間件
//auth 別名 editor是中間件的額外參數 在中間件類 handle加的
Route::get('/', function () { })->middleware([First::class, 'auth:editor']); 

取中間件別名
app/Http/Kernel.php裡面
protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,.....

排除中間件
中間件給一組路由但其中有不要用中間件的
Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', function(){});
    Route::get('/profile', function(){})->withoutMiddleware([EnsureTokenIsValid::class]);
});

一組路由都排除中間件
Route::withoutMiddleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/profile', function () { });
});

中間件組
多個中間件分組在一個鍵下 在App\Providers\RouteServiceProvider設定
protected $middlewareGroups = [
    'web' => [
        \App\Http\Middleware\EncryptCookies::class,...
在路由
Route::get('/', function(){})->middleware('web');
Route::middleware(['web'])->group(function(){});

排序中間件
極少數情況下，您可能需要中間件以特定順序執行，但在將它們分配給路由時無法控制它們的順序
在app/Http/Kernel.php 的
protected $middlewarePriority = [
    \Illuminate\Foundation\Http\Middleware\HandlePrecognitiveRequests::class,...

========================================
CSRF Cross Site Request Forgery偽造
定義“POST”、“PUT”、“PATCH”或“DELETE” 時

從 CSRF 保護中排除 URI
Illuminate\Foundation\Http\Middleware\VerifyCsrfToken的
protected $except = [ 'stripe/*','http://example.com/foo/*'...];

ajax的話  可以存在 HTMLmeta標記
jquery裡 加入header
$.ajaxSetup({
    headers: { 'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content') }
});
Laravel 將當前的 CSRF 存在一個加密的XSRF-TOKEN cookie 中。可以使用 cookie 值來設置X-XSRF-TOKEN請求標頭
Axios HTTP 庫，它將自動X-XSRF-TOKEN為您發送標頭。

========================================================
控制器 
https://artisan.page/

編寫控制器
php artisan make:controller UserController --api
php artisan make:controller PhotoController --resource
php artisan make:controller PhotoController --model=Photo --resource --requests

控制器裡設定使用中間件
public function __construct()
{
   $this->middleware('auth');
   this->middleware('log')->only('index');
   $this->middleware('subscribed')->except('store');
   
   //單個控制器定義內聯中間件的方法，不需定義整個中間件類
   $this->middleware(function (Request $request, Closure $next) {
        return $next($request);
    });
}

resource控制器
所有資源控制器操作都有一個 route name ex: photos.index
php artisan make:controller PhotoController --resource
php artisan make:controller PhotoController --api
resource路由
Route::resources([ 'photos' => PhotoController::class...]);
Route::apiResources([....

其他 get post等 有跟resource 一樣的路由 要加在recource 前面
Route::get(..
Route::resource(..

verb         uri                 action   route Name
GET        /photos	             index	  photos.index   
GET        /photos/create	     create	  photos.create  api(x)
POST       /photos	             store	  photos.store
GET        /photos/{photo}	     show	  photos.show    
GET        /photos/{photo}/edit	 edit	  photos.edit    api(x)
PUT/PATCH  /photos/{photo}	     update	  photos.update
DELETE     /photos/{photo}	     destroy  photos.destroy

軟刪除模型 要檢索到軟刪除的資料
在路由
Route::resource('photos', PhotoController::class)->withTrashed();

部分resource路由 only except
Route::resource('photos', PhotoController::class)->only([ 'index', 'show' ]);

Nested resource
照片資源可能有多個附加到照片的評論
Route::resource('photos.comments', PhotoCommentController::class);
/photos/{photo}/comments/{comment}

指定檢索子資源的哪個字段
Route::resource('photos.comments', PhotoCommentController::class)->scoped([
    'comment' => 'slug',
]);
路由會是
/photos/{photo}/comments/{comment:slug}


依賴注入和控制器
構造函數注入
public function __construct(
        protected UserRepository $users,
        
方法注入        
public function store(Request $request): RedirectResponse
{
=================================
HTTP 請求
Illuminate\Http\Request
處理的當前 HTTP 請求進行交互，檢索隨請求提交的輸入、cookie 和文件

與請求交互

accessing 請求
路由閉包或控制器方法上對類進行類型提示。傳入的請求實例將由 Laravel服務容器自動注入
Route::put('/user/{user}', [UserController::class, 'update']);
Request 先 路由參數 後
public function store(update $request,User $user): RedirectResponse
    {
        $name = $request->input('name');

請求路徑、主機和方法
$uri = $request->path(); foo/bar
$request->is('admin/*')  是否與route匹配
$request->routeIs('admin.*') 是否與命名路由匹配
$request->url()
$request->fullUrl()包含查詢字符串
$request->fullUrlWithQuery(['type' => 'phone']);與當前查詢url 合併
$request->host(); abc.joke.at.tw
$request->httpHost();abc.joke.at.tw
$request->schemeAndHttpHost();http://abc.joke.at.tw
$request->isMethod('post')
$request->header('X-Header-Name', 'default'); //沒有就用defalut
$request->bearerToken() 用於從Authorization報頭中檢索承載token
$request->ip();
$request->getAcceptableContentTypes();請求接受的content types
if ($request->accepts(['text/html', 'application/json']))

request 輸入
$request->all();
$request->collect();
$request->input('name', 'Sally');//沒有就用Sally
$request->input('products.0.name'); 
$request->input('products.*.name');
$request->query('name', 'Helen');

檢索 JSON 輸入值
JSON 請求
$name = $request->input('user.name');

通過動態屬性檢索輸入
$name = $request->name;

檢索輸入數據的一部分
$request->only(['username', 'password']);
$request->except(['credit_card']);

確定是否存在所有指定query key
if ($request->has(['name', 'email']))
$request->whenHas('name', function (string $input) {
    // 如果存在
}, function () {
    // 如果不存在
});

一個query key值是否出現在請求中並且不是空字符串
上面改成 filled whenFilled

是否缺少給定的query key
上面改成 missing whenMissing

修改query值
$request->merge(['votes' => 0]);

如果query key不存在 加上key value
$request->mergeIfMissing(['votes' => 0]);

檢索 Cookie
$value = $request->cookie('name');

舊輸入
在下一個請求期間保留來自一個請求的輸入

暫存Input到Session
$request->flash();
重定向
return redirect('form')->withInput();
return redirect()->route('user.create')->withInput();
檢索舊輸入
$username = $request->old('username');
blade顯示舊輸入
<input type="text" name="username" value="{{ old('username') }}">

文件
https://github.com/symfony/symfony/blob/6.0/src/Symfony/Component/HttpFoundation/File/UploadedFile.php
檢索上傳的文件
返回Illuminate\Http\UploadedFile類的一個實例
$file = $request->file('photo');
$file = $request->photo;

是否存在文件
if ($request->hasFile('photo'))

驗證成功上傳
if ($request->file('photo')->isValid()) {

文件路徑和擴展名
$path = $request->photo->path();
$extension = $request->photo->extension();

存儲上傳的文件 
相對於文件系統配置的根目錄的文件存儲路徑、文件名和磁盤名
$path = $request->photo->storeAs('images', 'filename.jpg', 's3');

配置可信代理
App\Http\Middleware\TrustProxies
配置可信主機
App\Http\Middleware\TrustHosts
====================================================
Response
Illuminate\Http\Response
return response('Hello World', 200)
                  ->header('Content-Type', 'text/plain');
返回Eloquent ORM模型和集合。 Laravel 會自動將模型和集合轉換為 JSON 響應，同時尊重模型的隱藏屬性：protected $hidden = ['password'];

將header附加到response
response($content)->withHeaders(['Content-Type' => $type,...]);

緩存控制中間件
Laravel 包含一個cache.headers中間件
https://blog.techbridge.cc/2017/06/17/cache-introduction/

將 Cookie 附加到response
$cookie = cookie('name', 'value', $minutes);
response('Hello World')->cookie($cookie);
想確保 cookie 與傳出響應一起發送
Cookie::queue('name', 'value', $minutes);
刪除 cookie
response('Hello World')->withoutCookie('name');

重定向
redirect('home/dashboard');
望將用戶重定向到他們以前的位置
back()->withInput();

重定向到命名路由 有參數
redirect()->route('profile', ['id' => 1]);
通過 Eloquent 模型填充參數
redirect()->route('profile', [$user]);

想自訂放置在路由參數中的值，你可以在路由參數定義中指定列 ( /profile/{id:slug}) 或者你可以覆蓋getRouteKeyEloquent 模型上的方法：
public function getRouteKey(): mixed {return $this->slug;}

重定向到控制器操作 有參數
return redirect()->action([UserController::class,'index']['id'=>1]);

重定向到外部域
return redirect()->away('https://www.google.com');

重定向並閃存session
return redirect('dashboard')->with('status', 'Profile updated!');
blade 
@if (session('status'))
    <div>{{ session('status') }} </div>
@endif

response 類型
需要控制響應的狀態和標頭，但還需要返回一個視圖
return response()->view('hello', $data, 200)->header('Content-Type', $type);

json
return response()->json(['name' => 'Abigail',]);

JSONP
return response()->json(['name' => 'Abigail''])->withCallback($request->input('callback'));

文件下載
// 文件路徑   用戶看到的文件名 標頭數組
return response()->download($pathToFile, $name, $headers);

下載不存入disk
use App\Services\GitHub;
return response()->streamDownload(function () {
    echo GitHub::api('repo')->contents()->readme('laravel', 'laravel')['contents'];
}, 'laravel-readme.md');

直接在瀏覽器中顯示文件
return response()->file($pathToFile, $headers); //header array

自訂response
通常，您應該從應用程序的服務提供者boot之一的方法中調用此方法
例如服務提供者：App\Providers\AppServiceProvider
public function boot(): void
{
   Response::macro('caps', function (string $value) {
     return Response::make(strtoupper($value));
   });
}
//使用
return response()->caps('foo');
============================
View
嵌套視圖目錄
return view('admin.profile', $data);
創建存在於給定視圖數組中的第一個視圖
return View::first(['custom.admin', 'admin'], $data);
視圖是否存在
if (View::exists('emails.customer'))

所有視圖共享數據
加到App\Providers\AppServiceProvider類中或生成一個單獨的服務提供者
public function boot(): void{ View::share('key', 'value');}

視圖組合器
https://hoohoo.top/blog/laravel-view-composer-introduction/
==============================
Blade 模板
顯示數據
return view('welcome', ['name' => 'Samantha']);
Hello, {{ $name }}
{{ time() }}

{{ }}語句會自動通過 PHP 的函數發送htmlspecialchars
不希望數據被轉義
{!! $name !!}

使用該符號@來通知 Blade 渲染引擎表達式應該保持不變
@{{ name }}

呈現 JSON
var app = <?php echo json_encode($array); ?>;
//把給定的對像或數組轉換為有效的 JavaScript 對象
var app = {{ Illuminate\Support\Js::from($array) }};
var app = {{ Js::from($array) }};
 
//多個js變數 用verbatim包起來
@verbatim
    <div class="container">
        Hello, {{ name }}.
    </div>
@endverbatim

blade指令

如果語句
@if (count($records) === 1)
    I have one record!
@elseif (count($records) > 1)
    I have multiple records!
@else
    I don't have any records!
@endif

@isset($records)
    // $records is defined and is not null...
@endisset
 
@empty($records)
    // $records is "empty"...
@endempty

身份驗證指令
@auth ... @endauth
@guest ... @endguest
@auth('admin')... @endauth
@guest('admin')... @endguest
 
 環境指令
@production...@endproduction
@env('staging')...@endenv
@env(['staging', 'production'])...@endenv

部分指令

開關語句
@switch($i)
    @case(1)
        @break
    @default
@endswitch

循環
@for ($i = 0; $i < 10; $i++)...{{ $i }}...@endfor

//foreach，您可以使用循環變量來獲取有關循環的有價值信息
// @if ($loop->first)  @if ($loop->last)
//嵌套循環 @if ($loop->parent->first) 上一層
//$loop->index	當前循環迭代的索引（從 0 開始）。
$loop->iteration	當前循環迭代（從 1 開始）。
$loop->remaining	循環中剩餘的迭代。
$loop->count	正在迭代的數組中的項目總數。
$loop->first	這是否是循環中的第一次迭代。
$loop->last	這是否是循環中的最後一次迭代。
$loop->even	這是否是循環中的偶數迭代。
$loop->odd	這是否是循環中的奇數迭代。
$loop->depth	當前循環的嵌套層級。
$loop->parent	在嵌套循環中時，父循環變量。
@foreach ($users as $user)... {{ $user->id }}...@endforeach
 
@forelse ($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users</p>
@endforelse
@while (true)...@endwhile

@continue  
@break 
@continue($user->type == 1)
@break($user->number == 5)

包含子視圖。對父視圖可用的所有變量都將對子視圖可用
@include('view.name', ['status' => 'complete'])

@php
    $counter = 1;
@endphp

===============================================================





