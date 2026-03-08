---
layout: post
title: "Building backend microservices with Go, Postgres, and Docker (part 2)"
categories: blog
---

In the [previous post]({% post_url 2026/2026-03-07-microservices-with-go-and-docker-part-1 %}), we set up the first Go microservice for our backend API. That service dealt with blog posts; today, we'll look at the microservice that handles comments. We'll mainly focus on using gRPC to have the two services talk to each other.

To start, I added a new table to the database schema to store comments placed on our posts:

```sql
CREATE TABLE comments (
    id               SERIAL PRIMARY KEY,
    post_id          INT NOT NULL,
    content_markdown TEXT NOT NULL,
    content_html     TEXT NOT NULL,
    author           TEXT NOT NULL,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT fk_comments FOREIGN KEY (post_id) REFERENCES posts(id)
);

CREATE INDEX ix_comments ON comments (post_id);
```

Note that for the sake of convenience, I'm having our two microservices share the same database; a more realistic scenario is both services having their own separate, independent data storage.

Next, I set up the comments Go microservice in basically the same way as I have the posts service, including a single http endpoint that lists the comments for a given post. Since this is effectively a repeat of the previous part, I won't go over all of that here again. Rather, I'm going to focus the rest of this post on using [gRPC](https://grpc.io/) to have the two services talk to each other. The posts service is going to host a gRPC server, and the comments service is going to implement a gRPC client.

The usecase is simple: when a new comment is submitted to the comments service, it's going to retrieve the corresponding post from the posts service (over gRPC), check if the post exists and allows comments to be placed, and if so, store the new comment in the database. Now, since in this toy example both services are sharing the same database, it doesn't make a whole lot of sense to do it this way, but let's imagine for the sake of argument that these services do not share a storage medium. In that case, gRPC is a reasonable way for having one service talk to another.

## Setting up the gRPC server

We'll create a folder called `common` inside of the `api` folder, and add our `posts.proto` file:

```protobuf
syntax = "proto3";

option go_package = "bw/blog-service/common/posts";

service PostService {
  rpc GetPost(GetPostRequest) returns (GetPostResponse) {}
}

message GetPostRequest {
    int32 postId = 1;
}

message GetPostResponse {
    optional Post post = 1;
}

message Post {
    int32 postId = 1;
    string title = 2;
    bool allowComments = 3;
}

```

Then, in the root of the posts service project, we'll create an empty `common/proto/posts` folder. We'll then run the following command to generate our protobuf Go code:

```bash
> protoc --proto_path="../common/" "posts.proto" \
    --go_out="common/proto/posts" --go_opt=paths=source_relative \
    --go-grpc_out="common/proto/posts" --go-grpc_opt=paths=source_relative
```

That produces two generated Go files:

![Protobuf generated code](/assets/img/blog/2026/03/protobuf-gen-code.png)

We'll also need to install the `google.golang.org/grpc` package.

We'll repeat this process for the comments service, so both services have their own copy of the generated code from the shared protobuf contract.

Next, we'll need to do some small refactoring on the posts service. Rather than having a single `application` struct that represents our app, I'll rename `application` to `appHttpServer` and introduce a new `appGrpcServer` to contain the gRPC server logic. It will have its own config, and will also receive a reference to the database connection so it can instantiate its own dependencies.

```go
type appGrpcServer struct {
    config appGrpcConfig
    db     *pgx.Conn
}

type appGrpcConfig struct {
    addr string
}
```

But first, let's implement our gRPC handler:

```go
type handlerGrpc struct {
    service Service
    proto.UnimplementedPostServiceServer
}
```

Here, we embed the `UnimplementedPostServiceServer` from the generated protobuf code, as well as a reference to our own `Service` interface, which contains our `ListPosts` and `GetPost` functions. On this type, we need to implement all of the functions specified in the generated protobuf code, which in our case is just the `GetPost` function from our protobuf specification:

![Protobuf generated unimplemented method](/assets/img/blog/2026/03/protobuf-unimplemented-func.png)

We'll implement this function as follows:

```go
func (h *handlerGrpc) GetPost(ctx context.Context, req *proto.GetPostRequest) (*proto.GetPostResponse, error) {
    post, err := h.service.GetPost(ctx, req.PostId)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return &proto.GetPostResponse{
                Post: nil,
            }, nil
        }

        slog.Error("error on service.GetPost", "error", err, "id", req.PostId)
        return nil, err
    }

    return &proto.GetPostResponse{
        Post: &proto.Post{
            PostId:        post.ID,
            Title:         post.Title,
            AllowComments: post.AllowComments,
        },
    }, nil
}
```

This function simply fetches the post from the database, and returns the appropriate protobuf response. It's very similar to the `GetPost` function on on the http handler.

With this in place, we can create a function for instantiating our gRPC handler:

```go
func NewHandlerGrpc(grpc *grpc.Server, svc Service) {
    handler := &handlerGrpc{
        service: svc,
    }

    proto.RegisterPostServiceServer(grpc, handler)
}
```

This function takes a gRPC server, as well as any other dependencies we might have (in our case, just our `Service` interface). It then registers itself with the gRPC server through another call to the generated protobuf code.

With our handler in place, let's go back and set up our gRPC server:

```go
func (app *appGrpcServer) run(ctx context.Context) error {
    ctx, cancel := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    lis, err := net.Listen("tcp", app.config.addr)
    if err != nil {
        return err
    }

    srv := grpc.NewServer()

    db := repo.New(app.db)

    postService := posts.NewService(db)
    posts.NewHandlerGrpc(srv, postService)

    serverErr := make(chan error, 1)

    go func() {
        slog.Info("grpc server: listening", "addr", app.config.addr)
        if err := srv.Serve(lis); err != nil {
            serverErr <- err
        }
    }()

    select {
    case err := <-serverErr:
        return err
    case <-ctx.Done():
        slog.Debug("grpc server: context cancelled")
    }

    slog.Info("grpc server: shutting down")
    timer := time.AfterFunc(10*time.Second, func() {
        slog.Debug("grpc server: force stop")
        srv.Stop()
    })
    defer timer.Stop()
    srv.GracefulStop()

    return nil
}
```

A lot of code, but as with the http server most of it is dedicated to handling graceful shutdown. We open a TCP connection to listen on the given address, instantiate a new gRPC server, instantiate our dependencies (the posts service), register our gRPC handler with said dependencies, and start listening.

Finally, we need to change our `main()` function, as we now need to spin up two different servers, so we can no longer simply block on running our http server.

```go
func main() {
    // [...]

    g, ctx := errgroup.WithContext(context.Background())

    // [...]

    g.Go(func() error {
        return appHttp.run(ctx, appHttp.init())
    })
    g.Go(func() error {
        return appGrpc.run(ctx)
    })

    if err := g.Wait(); err != nil {
        panic(err)
    }
}
```

Here, I use an error group to start both apps, which handles any errors that might occur in any one of them and cancels its context when the first error occurs.

## Calling the gRPC server

On the client side (the comments service), we have a copy of the same generated protobuf code, but we have a lot less code to write ourselves than on the server side.

We'll start by expanding our service implementation to depend on a gRPC client connection:

```go
type serviceImpl struct {
    repo     repo.Querier
    postGrpc *grpc.ClientConn
}

func NewService(repo repo.Querier, postGrpc *grpc.ClientConn) Service {
    return &serviceImpl{
        repo:     repo,
        postGrpc: postGrpc,
    }
}
```

With this, we can implement the following function for fetching comments:

```go
func (s *serviceImpl) ListComments(ctx context.Context, postId int32) ([]repo.Comment, error) {
    postClient := protogen.NewPostServiceClient(s.postGrpc)
    reqGrpc := &protogen.GetPostRequest{
        PostId: postId,
    }
    postResp, err := postClient.GetPost(ctx, reqGrpc)
    if err != nil {
        return nil, err
    }

    if postResp.Post == nil {
        return nil, errors.New("post not found")
    }

    return s.repo.ListComments(ctx, postId)
}
```

This function first calls the posts service over gRPC to see if the post in question exists, and only if it does, does it retrieve the comments from the database. A bit of a silly usecase, perhaps, but it's a simple way to illustrate gRPC in action and see if our connection is working.

Then, in our startup function, we instantiate a new gRPC client and pass it to our application:

```go
func main() {
    // [...]

    connPosts, err := grpc.NewClient(":9090", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        panic(err)
    }
    defer connPosts.Close()

    app := application{
        config:   cfg,
        db:       connDb,
        postGrpc: connPosts,
    }

    // [...]
}
```

Our application then passes it on to our comments service:

```go
func (app *application) init() http.Handler {
    // [...]

    commentService := comments.NewService(db, app.postGrpc)

    // [...]
}
```

Other than these changes, the comments service is essentially identical to the posts service as we set it up in the previous part.

With these changes now in place, let's start both services and see what happens...

```bash
> curl http://localhost:8181/posts/1/comments
[{"id":1,"post_id":1,"content_markdown":"hey you","content_html":"\u003cp\u003ehey you\u003c/p\u003e","author":"bw","created_at":"2026-03-08T15:17:21.771858+01:00"}]
```

Ok, great success! We can basically infer from this that the gRPC calls are working. Although, unlike with http requests, we don't currently see any of them show up in the logs. Let's see if we can change that.

## Logging on the gRPC server

As it turns out, the gRPC server has its own kind of middleware that you can inject into the request pipeline. Let's define a very simple logger, like so:

```go
func GrpcLogging(ctx context.Context, req any, i *grpc.UnaryServerInfo, h grpc.UnaryHandler) (any, error) {
    startTime := time.Now()

    m, err := h(ctx, req)

    elapsedTime := time.Since(startTime)
    slog.Info("grpc request", "method", i.FullMethod, "duration", elapsedTime)

    return m, err
}
```

Then, we change our gRPC server instantiation as follows:

```go
func (app *appGrpcServer) run(ctx context.Context) error {
    // [...]

    srv := grpc.NewServer(grpc.UnaryInterceptor(middleware.GrpcLogging))

    // [...]
}
```

And that's all! With this in place, we can see gRPC calls in our logs:

```bash
> go build -o server.exe .\cmd\ && .\server.exe
time=2026-03-08T15:39:15.328+01:00 level=INFO msg="app: starting"
time=2026-03-08T15:39:15.350+01:00 level=INFO msg="db: connected" host=localhost port=5432 database=blog user=pgroot
time=2026-03-08T15:39:15.350+01:00 level=INFO msg="http server: listening" addr=:8080        
time=2026-03-08T15:39:15.350+01:00 level=INFO msg="grpc server: listening" addr=:9090        
time=2026-03-08T15:42:05.307+01:00 level=INFO msg="grpc request" method=/PostService/GetPost duration=18.2778ms
```

Very nice.

## Posting a comment

Next, let's implement the http API method for posting comments to our blog posts. We'll cover handling of http POST requests and database INSERTs, and use our new gRPC service-to-service call to check if the post exists before accepting the comment.

We start by defining a type for the http request:

```go
package comments

type addCommentRequest struct {
    ContentMarkdown string `json:"content_markdown"`
}
```

We'll also introduce a custom error type, which we'll use later to distinguish between internal server errors and errors caused by the request input:

```go
package io

type RequestError struct {
    Message string
}

func (e *RequestError) Error() string {
    return e.Message
}
```

Then, we'll add the `AddComment` function to our service:

```go
type Service interface {
    ListComments(ctx context.Context, postId int32) ([]repo.Comment, error)
    AddComment(ctx context.Context, postId int32, req addCommentRequest) (repo.Comment, error)
}
```

We'll extract the logic for retrieving a post over gRPC into its own function:

```go
func getPost(ctx context.Context, postGrpc *grpc.ClientConn, postId int32) (*protogen.Post, error) {
    postClient := protogen.NewPostServiceClient(postGrpc)
    reqGrpc := &protogen.GetPostRequest{
        PostId: postId,
    }
    postResp, err := postClient.GetPost(ctx, reqGrpc)
    if err != nil {
        return nil, err
    }

    if postResp.Post == nil {
        return nil, &io.RequestError{Message: fmt.Sprintf("post with id %d not found", postId)}
    }

    return postResp.Post, nil
}
```

We'll then implement the `AddComment` function as follows:

```go
func (s *serviceImpl) AddComment(ctx context.Context, postId int32, req addCommentRequest) (repo.Comment, error) {
    post, err := getPost(ctx, s.postGrpc, postId)
    if err != nil {
        return repo.Comment{}, err
    }

    if !post.AllowComments {
        return repo.Comment{}, &io.RequestError{Message: fmt.Sprintf("post with id %d does not allow comments", postId)}
    }

    return s.repo.AddComment(ctx, repo.AddCommentParams{
        PostID:          postId,
        ContentMarkdown: req.ContentMarkdown,
        ContentHtml:     fmt.Sprintf("<p>%s</p>", req.ContentMarkdown), // TODO: convert markdown to HTML properly
        Author:          "unknown",                                     // TODO: get author from auth context
    })
}
```

There's a few TODOs in the code, but those are out of scope for now. This function fetches the post from the post service over gRPC, checks if it exists and allows comments, and if so, inserts a new comment into the database (using the sqlc generated repository). The newly inserted comment is then returned.

We'll need one final helper function for reading JSON from an http request:

```go
func ReadJSON(r *http.Request, data any) error {
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()
    return decoder.Decode(data)
}
```

After that, we can implement the handler for this method:

```go
func (h *handler) AddComment(w http.ResponseWriter, r *http.Request) {
    idStr := r.PathValue("post_id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        io.WriteBadRequest(w, "invalid value for 'post_id' provided")
        return
    }

    var req addCommentRequest
    if err := io.ReadJSON(r, &req); err != nil {
        slog.Error("error reading request body", "error", err)
        io.WriteBadRequest(w, "invalid request body provided")
        return
    }

    comment, err := h.service.AddComment(r.Context(), int32(id), req)
    if err != nil {
        if reqErr, ok := err.(*io.RequestError); ok {
            io.WriteBadRequest(w, reqErr.Message)
            return
        }

        slog.Error("error on service.AddComment", "error", err)
        io.WriteError(w, err)
        return
    }

    io.WriteJSON(w, http.StatusOK, &comment)
}
```

Here, we parse the request, consisting of the post ID from the path string and the comment from the request body, perform the request to the service, handle any errors, and return the newly added comment if everything was ok.

Now all that's left is registering our handler:

```go
func (app *application) init() http.Handler {
    // [...]

    mux.HandleFunc("POST /posts/{post_id}/comments", commentHandler.AddComment)

    // [...]
}
```

And with that, the comments service is good to go. Testing it in our API client of choice gives the expected result:

![API testing in Bruno](/assets/img/blog/2026/03/bruno-post-comment.png)

Excellent!

In the next post, we'll look at how to containerize our application with Docker.
