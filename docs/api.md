# API server
Generic library for building a LoxiLB API server.

# Usage
현재 API 서버는 HTTP, HTTPS모두 지원하며, Loxilb 실행시 -a 혹은 --api 옵션을 추가해서 실행이 가능하다.
API에 사용되는 옵션은 다음과 같다. 보안을 위해 HTTPS 옵션 --tls-key, --tls-certificate를 모두 주어야만 실행이 가능하다.

Currently, the API server supports both HTTP and HTTPS, and can be run by adding -a or --api options when running Loxilb. The options used in the API are as follows. For security purposes, HTTPS options --tls-key, --tls-certificate must be given to run.

```
      --host=            the IP to listen on (default: localhost) [$HOST]
      --port=            the port to listen on for insecure connections, defaults to a random value [$PORT]
      --tls-host=        the IP to listen on for tls, when not specified it's the same as --host [$TLS_HOST]
      --tls-port=        the port to listen on for secure connections, defaults to a random value [$TLS_PORT]
      --tls-certificate= the certificate to use for secure connections [$TLS_CERTIFICATE]
      --tls-key=         the private key to use for secure connections [$TLS_PRIVATE_KEY]
```

실제 사용하는 예시는 다음과 같다.

Examples of practical use are as follows.

```
 ./loxilb --tls-key=api/certification/server.key --tls-certificate=api/certification/server.crt --host=0.0.0.0 --port=8081 --tls-port=8091 -a
```

# API list
현재 API는 Load balancer 에 대한 Create, Delete, Read API가 있다.
Currently, the API has Create, Delete, and Read APIs for Load balancer.

| Method | URL | Role | 
|------|---|---|
| GET|/netlox/v1/config/loadbalancer/all | Get the load balancer information |
| POST|/netlox/v1/config/loadbalancer| Add the load balancer information to LoxiLB |
| DELETE|/netlox/v1/config/loadbalancer/externalipaddress/{IPaddress}/port/{#Port}/protocol/{protocol} | Delete the load balacer infomation from LoxiLB|

더 자세한 정보( Param, Body 등)은 Swagger문서를 참조한다.

See Swagger documentation for more information (Param, Body, etc.).

# LoxiLB API development guide
## API source Architecture
```
.
├── certification
│   ├── serverca.crt
│   └── serverkey.pem
├── cmd
│   └── loxilb_rest_api-server
│       └── main.go
├── ….
├── models
│   ├── error.go
│   ├── …..
├── restapi
│   ├── configure_loxilb_rest_api.go
│   ├── …..
│   ├── handler
│   │   ├── common.go
│   │   └──…..
│   ├── operations
│   │   ├── get_config_conntrack_all.go
│   │   └── ….
│   └── server.go
└── swagger.yml

```
* Except for the ./api/restapi/handler and ./api/certification directories, the rest of the contents are automatically created.
* Add the logic for the function to the handler directory.
* Add logic to file ./api/restapi/configure_loxilb_rest_api.go

1. Swagger.yml file update
```
paths:
  '/additional/url/{param}':
    get:
      summary: Test Swagger API Server.
      description: Check Swagger API server. This basic information or architecture is for the later applications.
      parameters:
        - name: param
          in: path
          required: true
          type: string
          format: string
          description: Description of the additional url
      responses:
        '204':
          description: OK
        '400':
          description: Malformed arguments for API call
          schema:
            $ref: '#/definitions/Error'
        '401':
          description: Invalid authentication credentials

```
* path.{Set path and parameter URL}.{get,post,etc RESTful setting}.{Description}
- {Set path and parameter URL}
Set the path used in the RESTful API.
It begins with "config/" and is defined as a sub-category from a large category.
Define the parameters using the symbol {param}. The parameters are defined in the description section.
- {get,post,etc RESTful setting}
Use get, post, delete, and patch to define queries, registrations, deletions, and modifications.
- {Description}
Summary description of API
Detailed description of API
Parameters
Set the name, path, etc.
Define the content of the response

2. Creating Additional Parts with Swagger
```
# alias swagger='docker run --rm -it  --user $(id -u):$(id -g) -e GOPATH=$(go env GOPATH):/go -v $HOME:$HOME -w $(pwd) quay.io/goswagger/swagger'
# swagger generate server
```

3. Development of Additional Partial Handlers
```
package handler

import (
    "fmt"

    "github.com/go-openapi/runtime/middleware"

    "testswagger.com/restapi/operations"
)

func ConfigAdditionalUrl(params operations.GetAdditionalUrlParams) middleware.Responder {
    /////////////////////////////////////////////
    //             Add logic Here              //
    ////////////////////////////////////////////.
    return &ResultResponse{Result: fmt.Sprintf("params.param : %s", params.param)}
}

```
* Select the logic required for the ConfigAdditionalUrl portion of the handler directory. The required parameters come from operations.GetAdditionalUrlParams.

4. Additional Partial Handler Registration
```
func configureAPI(api *operations.LoxilbRestAPIAPI) http.Handler {
    ...... 
    // Change it by putting a function here
    api.GetAdditionalUrlHandler = operations.GetAdditionalUrlHandlerFunc(handler.ConfigAdditionalUrl)
 ….
}
```
* if api.{REST}...The Handler form is automatically generated, where if nil is erased and a handler is added to the operation function.
*    In many cases, additional generation is not possible. In that case, you can add the function by entering it separately. The name of the function consists of a combination of Method, URL, and Parameter.

