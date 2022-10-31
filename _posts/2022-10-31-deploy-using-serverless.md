---
layout: post
title:  "Backend App using Go, Gin & Serverless Framework (Lambda)"
subtitle: "Serverless Framework is LIT YO!"
date:   2022-10-31 02:44:01
categories: [ci, cd, tool, serverless, aws, lambda, tutorials]
---

Let’s start with a straightforward introduction to the tech stack we’ll be using. 

I come from the NodeJS world. I recently started writing the backend with Go. I needed a web framework. Found out about gin - it’s pretty popular here in the Go world. 

I just had to set up AWS credentials & write the serverless & MakeFile - **Serverless Framework** handled the rest (setting up & deployed to Lambda) - I guess this is the best part of the tutorial. 

---

This tutorial is divided into three parts: 

**1. Creating a go-gin server.**

**2. Deploying it to Lambda using Serverless Framework.**

**3. Solving CORS error.**  

---
**1. Creating a Go-Gin Server**


This tutorial doesn’t come with a folder structure - it’ll just be a server.go file & two very basic routes. Straight to the point.  

1. First things first, set up the project & install gin. 
    
    ```go
    // Setup project. This is my repo on github. 
    go mod init github.com/yash-dxt/go-gin-tutorial
    
    // go get commands are used for installing dependencies. I'm installing gin.  
    go get -u github.com/gin-gonic/gin
    ```
    
2. Once you do this - you’ll have two files → *go.sum* & *go.mod*. The next step would be creating a server.go file. I have taken this code snippet and added my own */pong* endpoint.  
    
    ```go
    package main
    
    import (
    	"net/http"
    
    	"github.com/gin-gonic/gin"
    )
    
    func main() {
    	r := gin.Default()
    
    	r.GET("/ping", func(c *gin.Context) {
    		c.JSON(http.StatusOK, gin.H{
    			"message": "pong",
    		})
    	})
    
    	r.GET("/pong", func(c *gin.Context) {
    		c.JSON(http.StatusOK, gin.H{
    			"message": "dude wtf.",
    			"ping":    "hit on ping to get a pong",
    		})
    	})
    
    	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
    }
    ```
    
3. This would work fine on your local machine but when you push it up to lambda it gives you a timeout error (Internal Server Error) in the response. This is where I was stuck for quite some time. To solve this - I had to use another [library](https://github.com/apex/gateway). Let’s install the library using the command below. 
    
    ```go
    go get github.com/apex/gateway
    ```
    
    I read an environment variable which is *null* in local - if environment is “development“ - I run the app using gateway.listenAndServe method and if it’s not - I use the same old Run method. We’re running this on port 3000. 
    
    The last few lines become: 
    
    ```go
    
    	env := os.Getenv("GO_ENV")
    	addr := ":3000"
    	
    	if env == "development" {
    		gateway.ListenAndServe(addr, r)
    	} else {
    		r.Run(addr)
    	}
    }
    ```
    
4. I also wrote a shell script which - starts the server. You can run them as individual commands on the command line but here’s the shell script. 
    
    ```bash
    #!/usr/bin/env bash
    go build
    echo "go build is done..."
    go run server.go
    ```
    
---

**2. Deploying it to Lambda using Serverless Framework**


The first thing is you would want to check if you have AWS credentials set up or not. On most devices, to check this you can do:  

```bash
cat ~/.aws/credentials
```

Check if you have the aws_secret_access_key and the aws_access_key_id. If not - go to IAM in your AWS console & make one. 

1. Making the MakeFile. MakeFile would automate the commands we are supposed to use. The MakeFile is present below. 
    
    ```Makefile
    .PHONY: build deploy
    
    # First command builds up and puts it in a directory. 
    # It's essentially the go build command with a gotcha. 
    build:
    	env GOOS=linux go build -ldflags="-s -w" -o bin/handler ./server.go
    
    # Serverless takes care of you - it's serverless deploy command. 
    # Use: npm i -g serverless 
    # if you haven't downloaded serverless yet. 
    
    deploy: clean build
    	sls deploy --verbose
    ```
    
2. Making the serverless file. 
    
    ```yaml
    service: tutorial # The name you want to give your service. 
    
    #our provider is aws and we want to use go1.x (it's supported!!!)
    provider:
      name: aws
      runtime: go1.x
      region: us-east-1
    
    # our makefile has built the code & put it in the bin/ folder
    # instead of copying the whole thing - copy just that folder. 
    package:
     exclude:
       - ./**
     include:
       - ./bin/**
    #define two of our endpoints here. 
    # GET /ping and GET /pong is all we have. 
    # and an environment variable.
    functions:
      handler:
        handler: bin/handler
        events:
          - http:
              path: /ping
              method: get
          - http: 
              path: /pong
              method: get
        environment:
          GIN_MODE: development
    ```
    
3. Last command would be running the make command
    
    ```bash
    make deploy
    ```
    

Note: Keep switching the terminals as some of those commands work in Powershell & some of them work in Bash.