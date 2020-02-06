## Ứng dụng theo dõi cân nặng sử dụng Nodejs với PostgreSQL (P1)

Vừa trải qua kỳ nghỉ tết nguyên đán với quá nhiều bánh chưng và thịt heo, chắc hẳn mỗi chúng ta đều có sự biến động về cân nặng không nhiều thì ít. 

Vấn đề tăng cân là mối quan tâm của tất cả mọi người, tăng cân sẽ kèm theo vô số hệ lụy đằng sau như bệnh tật, thẩm mỹ,...

Với mục đích áp dụng công nghệ với đời sống, đưa khoa học vào thực tiễn, vừa học vừa thực hành. Trong bài viết này chúng ta sẽ xây dựng một ứng dụng để kiểm soát cân nặng của mình bằng việc sử dụng nodejs và postgreSQL.

## Tạo Nodejs project

Trước hết, hãy đảm bảo rằng trên máy tính của bạn đã được cài node và npm. Tiếp theo, tạo thư mục của dự án bằng câu lệnh sau

```
mkdir node-weight-tracker
cd node-weight-tracker
npm init -y
```

Với câu lệnh npm init sẽ tạo cho bạnh file package.json. Trong bài đọc này, chúng ta sẽ sử dụng framework để hỗ trợ code nodejs, có rất nhiều framework hỗ trợ code nodejs hiện nay, nhưng nội trong bài viết này, chúng ta sẽ sử dụng **hapi** framework. Dưới đấy là một số module mà chúng ta sẽ xử dụng trong project này

- hapi: Framework để code nodejs
- bell: plugin hỗ trợ đăng nhập cho bên thứ 3
- boom: plugin cho HTTP errors
- cookie: plugin cho cookie
- inert: plugin để quản lý file tĩnh
- joi: validate request và response data
- vision: plugin để rendering HTML tamplates
- dotenv: quản lý các config sử dụng biến môi trường
- ejs: Công cụ để code javascrip
- postgres: PostgresSQL client
- nodemon: tự động khởi động lại Nodejs application

Cài đặt framework và các module liên quan bằng câu lệnh sau:

```
npm install @hapi/hapi@19 @hapi/bell@12 @hapi/boom@9 @hapi/cookie@11 @hapi/inert@6 @hapi/joi@17 @hapi/vision@6 dotenv@8 ejs@3 postgres@1

npm install --save-dev nodemon@2
```

Tạo mới file với tên .env ở root của thư mục project của bạn và thêm vào config

```
# Host configuration
PORT=8080
HOST=localhost
```

Tiếp theo, tạo các thư mục con bên trong thư mục project của bạn để có cấu trúc project như sau:

```
> node_modules
> src
   > assets
   > plugins
   > routes
   > templates
.env
package-lock.json
package.json
```

# Create "Hello World" với hapi

Trong thư mục src, tạo file index.js với nội dung sau;

```
"use strict";

const dotenv = require( "dotenv" );
const Hapi = require( "@hapi/hapi" );

const routes = require( "./routes" );

const createServer = async () => {
  const server = Hapi.server( {
    port: process.env.PORT || 8080,
    host: process.env.HOST || "localhost"
  } );

  server.route( routes );

  return server;
};

const init = async () => {
  dotenv.config();
  const server = await createServer();
  await server.start();
  console.log( "Server running on %s", server.info.uri );
};

process.on( "unhandledRejection", ( err ) => {
  console.log( err );
  process.exit( 1 );
} );

init();
```

Ngó lại đoạn code trên 1 chút, các bạn có thể thấy function init use _dotenv_ để đọc config từ file .env mà chúng ta đã tạo trước đó, tạo web server, khởi động web server, và mở cổng 8080. function createSerrver() khởi tạo một đối tượng của hapi dựa trên port và host mà chúng ta config trong .env. Và đăng ký một router trong routes module.

Cuối cùng là event unhandledRejection để xử lý khi có exception xảy ra, nó sẽ ghi log lại lỗi và shuts down server.

Hàm createServer() chúng ta có gọi đến routes, thì tới đây, bạn cần định nghĩa routes cho nó. Tạo file src/routes/index.js và thêm đoạn code sau:

```
"use strict";

const home = {
  method: "GET",
  path: "/",
  handler: ( request, h ) => {
    return "hello world!";
  }
};

module.exports = [ home ];
```

Với code trên, chúng ta định nghĩa route home, trả về dòng chữ "hello world" qua phương thức GET

Mở file package.json và tìm dòng scripts sau đó thêm dòng sau vào 

```
"dev": "nodemon --watch src -e ejs,js src/index.js",
```
Tiếp theo là mở command line và chạy câu lệnh

```
npm run dev
```
mở trình duyệt của bạn và đăng nhập vào link (http://localhost:8080)[http://localhost:8080] bạn sẽ thấy dòng chữ "hello world" được in trên màn hình. Quay lại file index.js trong routes, và thử thay đổi text hello world bằng một text khác xem, trình duyệt của bạn cũng được cập nhật ngay lập tức, đây là công dụng của nodemon nhé.

# Config PostgreSQL 

Trước tiên, hãy cài đặt PostgreSQl server bằng docker với câu lệnh sau:

```
docker pull postgres:latest
docker run -d --name measurements -p 5432:5432 -e 'POSTGRES_PASSWORD=p@ssw0rd42' postgres
```

Thêm config cho PostgreSQL trong file .env

```
# Postgres configuration
PGHOST=localhost
PGUSERNAME=postgres
PGDATABASE=postgres
PGPASSWORD=p@ssw0rd42
PGPORT=5432
```

## Connect to PostGreSQL

Thông thường, để sử dụng được SQL chúng ta cần tạo bảng, khởi tạo dữ liệu. Có một cách để thực hiện việc này là tạo ra script. Ở đây chúng ta sử dụng Nodejs để xây dựng schema cần thiết cho project

Tạo mới 1 folder với tên tools trong root project của bạn, trong folder này thì tạo thêm file initdb.js, thêm đoạn code sau vào file vừa được tạo

```
"use strict";

const dotenv = require( "dotenv" );
const postgres = require( "postgres" );

const init = async () => {
  // read environment variables
  dotenv.config();

  try {
    // connect to the local database server
    const sql = postgres();

    console.log( "dropping table, if exists..." );
    await sql`DROP TABLE IF EXISTS measurements`;

    console.log( "creating table..." );
    await sql`CREATE TABLE IF NOT EXISTS measurements (
      id INT NOT NULL PRIMARY KEY GENERATED ALWAYS AS IDENTITY
      , user_id varchar(50) NOT NULL
      , measure_date date NOT NULL
      , weight numeric(5,1) NOT NULL
    )`;

    await sql.end();
  } catch ( err ) {
    console.log( err );
    throw err;
  }
};

init().then( () => {
  console.log( "finished" );
} ).catch( () => {
  console.log( "finished with errors" );
} );
```

Cuối cùng là update scripts section trong file package.json

```
"initdb": "node tools/initdb.js",
```
Bây giờ bạn có thể lệnh _npm run innitdb_ để thực thực hiện file initdb.js mà chúng ta vừa khởi tạo. Khi cậu lệnh này chạy xong, bẹ sẽ thấy dòng chữ finised trên màn console. và table mới với tên measurements trong database . Bấy kỳ khi nào bạn muốn reset DB thì hãy chạy lại câu lệnh trên.

## Add authentication to Node and hapi

Trước tiên, bạn cần tạo một ứng dụng trên trang (https://developer.okta.com/)[https://developer.okta.com/] Sau khi hoàn tất việc đăng ký và tạo ứng dụng trên okta. update file .env với config sau

```
HOST_URL=http://localhost:8080
COOKIE_ENCRYPT_PWD=superAwesomePasswordStringThatIsAtLeast32CharactersLong!
NODE_ENV=development

# Okta configuration
OKTA_ORG_URL=https://{yourOrgUrl}
OKTA_CLIENT_ID={yourClientId}
OKTA_CLIENT_SECRET={yourClientSecret}
```

Nhớ cập nhật thông tin client của bạn vào nhé.

## Secure nodejs application

Sau khi bạn có okta account và applications. Tiếp theo cần tiến hành config để ứng dụng của chúng ta chỉ cho phép user nào đăng nhập vào mới có thể sử dụng. 

Tạo mới file src/plugins/auth.js, thêm dòng code

```
"use strict";

const bell = require( "@hapi/bell" );
const cookie = require( "@hapi/cookie" );

const isSecure = process.env.NODE_ENV === "production";

module.exports = {
  name: "auth",
  version: "1.0.0",
  register: async server => {

    await server.register( [ cookie, bell ] );

    // configure cookie authorization strategy
    server.auth.strategy( "session", "cookie", {
      cookie: {
        name: "okta-oauth",
        path: "/",
        password: process.env.COOKIE_ENCRYPT_PWD,
        isSecure // Should be set to true (which is the default) in production
      },
      redirectTo: "/authorization-code/callback", // If there is no session, redirect here
    } );

    // configure okta oauth strategy
    server.auth.strategy( "okta", "bell", {
      provider: "okta",
      config: { uri: process.env.OKTA_ORG_URL },
      password: process.env.COOKIE_ENCRYPT_PWD,
      isSecure,
      location: process.env.HOST_URL,
      clientId: process.env.OKTA_CLIENT_ID,
      clientSecret: process.env.OKTA_CLIENT_SECRET
    } );

    // set the default authorization strategy for all routes
    server.auth.default( "session" );

    // Hook into onPreResponse event to add authentication info to every view
    server.ext( "onPreResponse", ( request, h ) => {
      if ( request.response.variety === "view" ) {
        const auth = request.auth.isAuthenticated ? {
          isAuthenticated: true,
          isAnonymous: false,
          email: request.auth.artifacts.profile.email,
          firstName: request.auth.artifacts.profile.firstName,
          lastName: request.auth.artifacts.profile.lastName
        } : {
          isAuthenticated: false,
          isAnonymous: true,
          email: "",
          firstName: "",
          lastName: ""
        };
        request.response.source.context.auth = auth;
      }
      return h.continue;
    } );
  }
};
```

Tiếp theo là file src/plugins/index.js

```
"use strict";

const Inert = require( "@hapi/inert" );
const Vision = require( "@hapi/vision" );
const ejs = require( "ejs" );

const auth = require( "./auth" );

module.exports = {
  register: async server => {
    await server.register( [ Inert, Vision, auth ] );

    // configure view templates
    server.views( {
      engines: { ejs },
      relativeTo: __dirname,
      path: "../templates",
      layout: true
    } );
  }
};
```
Phần 1 này tạm thời dừng lại ở đây đã, trong phần tới chúng ta sẽ tiếp tục xây dựng ứng dụng với việc tạo template, c
