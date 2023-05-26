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






