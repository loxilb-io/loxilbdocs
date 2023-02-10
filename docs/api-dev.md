# loxilb API Development Guide

For building and extending LoxiLB API server.

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

