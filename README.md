# Sample Laravel application to demonstrate REST APIs with JWT authentication

## License

MIT

## Start up the environment

Start Docker (or Docker Desktop).  If using Docker Desktop, open up the window and go into the Containers view.

**Bring up the Docker containers for MariaDB and phpMyAdmin:**
```console
docker-compose up -d
```

**Start the development web server in a separate window:**
```console
php artisan serve
```

**Point a browser to (http://localhost:10000) and enter the following, then click "Log in":**
```
Server: mariadbtest
Username: root
Password: 123TeSt321
```

**Click on the SQL tab, enter the following command, and click "Go" (or, you can just run ./mysql.sh and enter/run this command inside the mysql console):**
```sql
create database laravel_jwt_rest_api_test;
```

You should now see the laravel_jwt_rest_api_test database in the left panel (you may need to refesh the page).

**Run the database migration in the project folder:**
```console
php artisan migrate
```

Back in phpMyAdmin, you should be able to expand the laravel_jwt_rest_api_test database in the left panel and see a number of tables which were created.

## Test the User APIs

**Test new user registration:**
```console
curl -XPOST -F "name=John Doe" -F "email=john@example.com" -F "password=admin123" -F "password_confirmation=admin123" "http://localhost:8000/api/auth/register"
```

Should return JSON similar to the following:
```
{"message":"User successfully registered","user":{"name":"John Doe","email":"john@example.com","updated_at":"2022-07-18T23:59:35.000000Z","created_at":"2022-07-18T23:59:35.000000Z","id":1}}
```

**Test user login:**
```console
curl -XPOST -F "email=john@example.com" -F "password=admin123" "http://localhost:8000/api/auth/login"
```

Should return JSON similar to the following (token is redacted here):
```
{"access_token":"<redacted>","token_type":"bearer","expires_in":3600,"user":{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}}
```

**Test user profile (replace <redacted> with the actual token from the response to the login request):**
```console
curl -XGET -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/user-profile"
```

Should return JSON similar to the following:
```
{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}
```

**Test JWT token refresh (replace <redacted> with the actual token from the response to the login request):**
```console
curl -XPOST -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/refresh"
```

Should return JSON similar to the following (new token is redacted here):
```
{"access_token":"<redacted>","token_type":"bearer","expires_in":3600,"user":{"id":1,"name":"John Doe","email":"john@example.com","email_verified_at":null,"created_at":"2022-07-18T23:59:35.000000Z","updated_at":"2022-07-18T23:59:35.000000Z"}}
```

**Test logout (replace <redacted> with the actual token from the response to the last login or refresh request):**
```console
curl -XPOST -H "Authorization: Bearer <redacted>" "http://localhost:8000/api/auth/logout"
```

Should return JSON similar to the following:
```
{"message":"User successfully signed out"}
```

## Test the Post APIs

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
