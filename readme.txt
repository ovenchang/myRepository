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

中間件
檢查和過濾進入應用程序的 HTTP 請求  是否已通過身份驗證 日誌記錄
身份驗證和 CSRF 保護的中間件。這些中間件都位於該app/Http/Middleware目錄中

定義中間件
php artisan make:middleware EnsureTokenIsValid
新增 app/Http/Middleware/EnsureTokenIsValid

//handle處理
public function handle(Request $request, Closure $next): Response
{
    if ($request->input('token') !== 'my-secret-token') {
        return redirect('home');
    }
     return $next($request);
 }

在應用程序處理請求之前執行
public function handle(Request $request, Closure $next): Response
{
        // 處理
        return $next($request);
}

在應用程序處理請求後執行
public function handle(Request $request, Closure $next): Response
{
        $response = $next($request);
        // 處理
        return $response;
}

註冊中間件

全局
app/Http/Kernel.php類的屬性中列出中間件

為路由分配中間件
Route::get('/', function () { })->middleware([First::class, 'auth']); //auth 別名

取中間件別名
app/Http/Kernel.php裡面
protected $middlewareAliases = [
    'auth' => \App\Http\Middleware\Authenticate::class,.....

中間件給一組路由但其中有不要用中間件的
Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/', function(){});
    Route::get('/profile', function(){})->withoutMiddleware([EnsureTokenIsValid::class]);
});






