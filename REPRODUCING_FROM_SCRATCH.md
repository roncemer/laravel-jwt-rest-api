# Reproducing this project from scratch

## Create a Laravel app; start MariaDB AND phpMyAdmin under Docker; set up JWT authentication; create user registration, login, profile, refresh and logout APIs

This section is adapted from this article: (https://www.positronx.io/laravel-jwt-authentication-tutorial-user-login-signup-api/)

**Create a new Laravel project, and get into its top-level folder:**

```console
composer create-project laravel/laravel laravel-jwt-rest-api --prefer-dist
cd laravel-jwt-rest-api
```

**Start Docker (or Docker Desktop).  If using Docker Desktop, open up the window and go into the Containers view.**

**Create a docker-compose.yml file with the following contents:**
```
version: "3"

services:

  mariadb:
    image: mariadb:10.6
    restart: always
    mem_limit: 2g
    container_name: mariadbtest
    environment:
      MYSQL_ROOT_PASSWORD: 123TeSt321
      TERM: xterm
    #volumes:
    #  - ./volumes/var/lib/mysql:/var/lib/mysql
    #  - ./volumes/etc/mysql/conf.d:/etc/mysql/conf.d

    ports:
      - "3306:3306"

    # --skip-name-resolve
    # avoids: ""[Warning] IP address '172.17.0.60' could not be resolved: Name or service not known""
    command: mysqld --skip-name-resolve

  phpmyadmin:
    image: phpmyadmin
    restart: always
    ports:
      - 10000:80
    environment:
      - PMA_ARBITRARY=1
```

**OPTIONAL: Create a mysql.sh file with the following contents, and chmod it to 755:**
```
#!/bin/bash

mysql -h 127.0.0.1 --user=root --password=123TeSt321
```

With this script, you can always get into MariaDB (as long as it's running) as the root user.

**Start the Docker containers:**
```console
docker-compose up -d
```

**To use phpMyAdmin, point a browser to (http://localhost:10000) and enter the following, then click "Log in":**
```
Server: mariadbtest
Username: root
Password: 123TeSt321
```

**Click on the SQL tab, enter the following command, and click "Go" (or, you can just run ./mysql.sh and enter/run this command inside the mysql console):**
```sql
create database laravel_jwt_rest_api_test;
```

**Edit the .env file and adjust the following settings to match these values:**
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel_jwt_rest_api_test
DB_USERNAME=root
DB_PASSWORD=123TeSt321
```

You should now see the laravel_jwt_rest_api_test database in the left panel of phpMyAdmin (you may need to refesh the page).

**Run the database migration in the project folder:**
```console
php artisan migrate
```

Back in phpMyAdmin, you should be able to expand the laravel_jwt_rest_api_test database in the left panel and see a number of tables which were created.

**Install the jwt-auth package:**
```console
composer require tomfordrumm/jwt-auth:dev-develop
```

NOTE: With Laravel v9.19.0, the tymon/jwt-auth Composer module requires a different version of Illuminate than that which is included with Laravel, so it cannot be installed.  Most articles online say that you should use tymon/jwt-auth, but since it's not compatible, you have to use this alternate package instead.  This is just a development version of tymon/jwt-auth which has been modified to work with the latest Laravel release.

**Edit config/app.php.  Add the following entry at the end of the providers array:**
```
  Tymon\JWTAuth\Providers\LaravelServiceProvider::class,
```
**Add the following entries at the end of the aliases array:**
```
  'JWTAuth' => Tymon\JWTAuth\Facades\JWTAuth::class,
  'JWTFactory' => Tymon\JWTAuth\Facades\JWTFactory::class,
```

**Publish the package's configuration, create a secret, and check that it was saved correctly:**
```console
php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
php artisan jwt:secret
grep JWT_SECRET .env
```

**Edit app/Models/User.php and replace its entire contents with the following:**
```
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Tymon\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{
    use HasFactory, Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $fillable = [
        'name',
        'email',
        'password',
    ];

    /**
     * The attributes that should be hidden for arrays.
     *
     * @var array
     */
    protected $hidden = [
        'password',
        'remember_token',
    ];

    /**
     * The attributes that should be cast to native types.
     *
     * @var array
     */
    protected $casts = [
        'email_verified_at' => 'datetime',
    ];
    
    /**
     * Get the identifier that will be stored in the subject claim of the JWT.
     *
     * @return mixed
     */
    public function getJWTIdentifier() {
        return $this->getKey();
    }

    /**
     * Return a key value array, containing any custom claims to be added to the JWT.
     *
     * @return array
     */
    public function getJWTCustomClaims() {
        return [];
    }    
}
```


**Edit config/auth.php making the following changes:**
  **Under 'defaults', change the value for *'guard'* from *'web'* to *'api'*.**
  **Under *'guards'*, add the following section under the *'web'* section:**
```
        'api' => [
            'driver' => 'jwt',
            'provider' => 'users',
            'hash' => false,
        ],
```

**Build the authentication controller:**
```console
php artisan make:controller Api/AuthController
```


**Edit app/Http/Controllers/Api/AuthController.php and replace its entire contents with the following:**
```
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use App\Models\User;
use Validator;

class AuthController extends Controller
{
    /**
     * Create a new AuthController instance.
     *
     * @return void
     */
    public function __construct() {
        $this->middleware('auth:api', ['except' => ['login', 'register']]);
    }

    /**
     * Get a JWT via given credentials.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function login(Request $request){
    	$validator = Validator::make($request->all(), [
            'email' => 'required|email',
            'password' => 'required|string|min:6',
        ]);
        if ($validator->fails()) {
            return response()->json($validator->errors(), 422);
        }
        if (! $token = auth()->attempt($validator->validated())) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }
        return $this->createNewToken($token);
    }

    /**
     * Register a User.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function register(Request $request) {
        $validator = Validator::make($request->all(), [
            'name' => 'required|string|between:2,100',
            'email' => 'required|string|email|max:100|unique:users',
            'password' => 'required|string|confirmed|min:6',
        ]);
        if($validator->fails()){
            return response()->json($validator->errors()->toJson(), 400);
        }
        $user = User::create(array_merge(
                    $validator->validated(),
                    ['password' => bcrypt($request->password)]
                ));
        return response()->json([
            'message' => 'User successfully registered',
            'user' => $user
        ], 201);
    }

    /**
     * Log the user out (Invalidate the token).
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function logout() {
        auth()->logout();
        return response()->json(['message' => 'User successfully signed out']);
    }

    /**
     * Refresh a token.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function refresh() {
        return $this->createNewToken(auth()->refresh());
    }

    /**
     * Get the authenticated User.
     *
     * @return \Illuminate\Http\JsonResponse
     */
    public function userProfile() {
        return response()->json(auth()->user());
    }

    /**
     * Get the token array structure.
     *
     * @param  string $token
     *
     * @return \Illuminate\Http\JsonResponse
     */
    protected function createNewToken($token){
        return response()->json([
            'access_token' => $token,
            'token_type' => 'bearer',
            'expires_in' => auth()->factory()->getTTL() * 60,
            'user' => auth()->user()
        ]);
    }
}
```

**Edit routes/api.php and replace its entire contents with the following:**
```
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\AuthController;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/

Route::group([
    'middleware' => 'api',
    'prefix' => 'auth'
], function ($router) {
    Route::post('/login', [AuthController::class, 'login']);
    Route::post('/register', [AuthController::class, 'register']);
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::post('/refresh', [AuthController::class, 'refresh']);
    Route::get('/user-profile', [AuthController::class, 'userProfile']);    
});
```

**Start the development web server in a separate terminal window or tab:**
```console
php artisan serve
```

**Test new user registration:**
```console
curl -XPOST -F "name=John Doe" -F "email=john@example.com" -F "password=admin123" -F "password_confirmation=admin123" "http://localhost:8000/api/auth/register"
```
Should return JSON similar to the following:
```
{"message":"User successfully registered","user":{"name":"John Doe","email":"john@example.com","updated_at":"2022-07-18T23:59:35.000000Z","created_at":"2022-07-18T23:59:35.000000Z","id":1}}
```

Test user login:
```console
curl -XPOST -F "email=john@example.com" -F "password=admin123" "http://localhost:8000/api/auth/login"
```
Should return JSON similar to the following (token is redacted here):
```
{"access_token":"<redacted>","token_type":"bearer","expires_in":3600,"user":{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}}
```

Test user profile (replace <redacted> with the actual token from the response to the login request):
```console
curl -XGET -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/user-profile"
```
Should return JSON similar to the following:
```
{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}
```

Test JWT token refresh (replace <redacted> with the actual token from the response to the login request):
```console
curl -XPOST -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/refresh"
```
Should return JSON similar to the following (new token is redacted here):
```
{"access_token":"<redacted>","token_type":"bearer","expires_in":3600,"user":{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}}
```

Test logout (replace <redacted> with the actual token from the response to the last login or refresh request):
```console
curl -XPOST -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/logout"
```
Should return JSON similar to the following:
```
{"message":"User successfully signed out"}
```

## Create tables; create REST APIs for performing CRUD (Create/Read/Update/Delete) operations on them

This section is adapted from this article: (https://larainfo.com/blogs/laravel-9-rest-api-crud-tutorial-example)

**Create some classes:**
```console
php artisan make:model Post -m
php artisan make:request StorePostRequest
php artisan make:resource PostResource
php artisan make:controller Api/PostController --model=Post
```

**Edit *database/migrations/2022_07_19_002237_create_posts_table.php* (note that the date/time will be different, so you'll have to find the file yourself) and replace its entire contents with the following:**
```
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('description');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('posts');
    }
};
```

**Edit app/Models/Post.php and replace its entire contents with the following:**
```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;
    protected $fillable = ['title', 'description'];
}
```

**Run the database migration:**
```console
  php artisan migrate
```

**Edit app/Http/Requests/StorePostRequest.php and replace its entire contents with the following:**
```
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     *
     * @return bool
     */
    public function authorize()
    {
        return true;
    }

    /**
     * Get the validation rules that apply to the request.
     *
     * @return array
     */
    public function rules()
    {
        return [
            'title' => ['required', 'max:70'],
            'description'  => ['required']
        ];
    }
}
```

**Edit app/Http/Controllers/Api/PostController.php and replace its entire contents with the following:**
```
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function __construct() {
        $this->middleware('auth:api', ['except' => []]);
    }

     /**
     * Display a listing of the resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function index()
    {
        $posts = Post::all();
        return PostResource::collection($posts);
    }

    /**
     * Show the form for creating a new resource.
     *
     * @return \Illuminate\Http\Response
     */
    public function create()
    {
        //
    }

    /**
     * Store a newly created resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    public function store(StorePostRequest $request)
    {
        $posts = Post::create($request->all());
        
        return new PostResource($posts);
    }

    /**
     * Display the specified resource.
     *
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function show(Post $post)
    {
        return new PostResource($post);
    }

    /**
     * Show the form for editing the specified resource.
     *
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function edit(Post $post)
    {
        //
    }

    /**
     * Update the specified resource in storage.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function update(StorePostRequest $request, Post $post)
    {
        $post->update($request->all());
        
        return new PostResource($post);
    }

    /**
     * Remove the specified resource from storage.
     *
     * @param  \App\Models\Post  $post
     * @return \Illuminate\Http\Response
     */
    public function destroy(Post $post)
    {
        $post->delete();

        return response(null, 204);
    }
}
```

**Edit routes/api.php.  Add the following line at the top, under the other "use" lines:**
```
use App\Http\Controllers\Api\PostController;
```
**Add the following lines at the end of the same file:**
```
Route::group([
    'middleware' => 'api',
], function ($router) {
    Route::apiResource('posts', PostController::class);
});
```

**Log in:**
```console
curl -XPOST -F "email=john@example.com" -F "password=admin123" "http://localhost:8000/api/auth/login"
```
Save a copy of the token to make it easier to copy and paste it into subsequent curl commands.

**Create a post (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XPOST -H "Authorization: Bearer <redacted>" -F "title=Laravel 9 REST API" -F "description=Lorem ipsum blah blah blah." "http://localhost:8000/api/posts"
```
Should return JSON similar to the following:
```
{"data":{"title":"Laravel 9 REST API","description":"Lorem ipsum blah blah blah.","updated_at":"2022-07-19T01:12:06.000000Z","created_at":"2022-07-19T01:12:06.000000Z","id":1}}
```

**Get all posts (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XGET -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/posts"
```
Should return all posts, in JSON format.  If you've created multiple posts, they should all be there.

**Get a single post (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XGET -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/posts/1"
```

**Update a post (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XPOST -H "Authorization: Bearer <redacted>" -F "_method=PUT" -F "title=Laravel 9 New Features" -F "description=Test." "http://localhost:8000/api/posts/1"
```

**Delete a post (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XDELETE -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/posts/1"
```

## Stopping everything

**Stop the development web server (php artisan serve) using *Ctrl+C*.**

**Stop the docker containers:**
```console
docker-compose down
```

