---
layout: post
title: How web-servers parallelise request load?
---

I recently started thinking about the differences in implementation of different Ruby web servers (Unicorn, Passenger, Puma), and what it means to implement a single server process, a server with multiple threads, and a server with multiple processes. It makes sense conceptually to horizontally scale using threads or processes depending on whether your application is thread-safe or not.

I was especially curious about how web-servers utilise multiple processes to spread request load, after coming across Einhorn to run multiple web-server processes at work. So I decided to implement a strawman version of different server types in Ruby to better understand the differences between them.

The client code for each implementation does not change, but I’ll share it here in case you’re looking to try it out:


    # client.rb
    # TCP Client
    require 'socket'

    hostname = 'localhost'
    port = 5678
    server = TCPSocket.new(hostname, port)

    while line = server.gets
      puts line
    end

    # close the socket
    server.close

The client creates a new socket and connects to the server address by specifying the `hostname` and `port`. The client can now call `gets` to read data sent by the server and simply print the line.

The server implementation changes based on the approach used and based on whether request parallelisation is done using threads or processes.

## Single-process server

This server implementation has a single process that listens on the socket, waits for a client to connect to the socket, and accepts an incoming connection. The server writes the current time to the socket and closes the connection when done. In this case when multiple requests arrive, the OS Kernel keeps them in an internal queue and the server process picks requests from the queue, processing them one at a time.


    # server.rb
    # TCP Server: Single threaded
    require 'socket'

    server = TCPServer.open(5678)
    loop do
      connection = server.accept
      connection.puts "The time is #{Time.now}"
      connection.close
    end

### Output
The command `lsof` shows the list of network connections (`-i`) that listen to port number `5678` . Starting the server and running this command shows that there is a single process listening to the port, which is what we expect for the implementation above.


    # start the server
    Ankitas-MBP:websockets ankitagupta$ ruby server.rb

    # run the client in a new tab
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    The time is 2020-07-24 09:55:34 +0800

    Ankitas-MBP:websockets ankitagupta$ lsof -P -i :5678 | grep LISTEN
    ruby    43082 ankitagupta   10u  IPv6 0xd05968e1cba9cb47      0t0  TCP *:5678 (LISTEN)

## Multi-threaded server

In the previous implementation, the server is blocked by each incoming request and processes concurrent requests serially. To speed things up a bit, the server below accepts incoming connections from the client, and passes the execution responsibility to a new thread. This frees the server to accept the next incoming connection sooner, while allowing the thread to do most of the work in parallel.


    # server.rb
    # TCP Server: Multi-threaded
    require 'socket'

    server = TCPServer.open(5678)
    loop do
      connection = server.accept

      # Start a new thread to handle any incoming connection
      Thread.start(connection) do |c|
        c.puts("The time is #{Time.now}")
        c.close
      end
    end

In practice, multi-threaded servers are more nuanced and typically use a thread pool to maintain a limited number of threads because spinning up a new thread per incoming connection is wasteful. Each incoming request is then handled by one of the available threads in the pool.

### Output
There is still a single server listening to the incoming connections on port `5678`.


    # start the server
    Ankitas-MBP:websockets ankitagupta$ ruby server.rb

    # run the client in a new tab
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    The time is 2020-07-24 09:53:10 +0800

    Ankitas-MBP:websockets ankitagupta$ lsof -P -i :5678 | grep LISTEN
    ruby    43031 ankitagupta   10u  IPv6 0xd05968e1cba9e3c7      0t0  TCP *:5678 (LISTEN)


## Multi-process server

The implementation below uses `Process.fork` to create new child processes, and each new process ends up listening to the same ports as the parent process. The OS schedules one of the processes listening to the port, allowing for concurrent connections to be handled by any one of the process, including the parent process.


    # server.rb
    # TCP Server: Multiple processes
    require 'socket'

    server = TCPServer.open(5678)
    session = server.accept
    session.puts "Server id: parent, The time is #{Time.now}"
    session.close
    # child processes
    2.times do |count|
      Process.fork do
        while session = server.accept
          session.puts "Server id: child #{count}, The time is #{Time.now}"
          session.close
        end
      end
    end

    # Wait for any child process with the same group ID as the calling process to exit
    Process.wait

Once again, actual multi-process server implementations and socket sharing libraries such as Einhorn provide other niceties such as spinning a fixed number of processes, spawning a new process if a process is terminated unexpectedly, and completing existing requests before shutting down the server.

### Output
The `lsof` command shows that there are three processes listening to incoming connections on the port - the parent process and two child processes created in the loop above by calling `Process.fork`.


    # start the server
    Ankitas-MBP:websockets ankitagupta$ ruby server.rb

    # run the client multiple times in a new tab
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: child 0, The time is 2020-07-24 09:57:19 +0800
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: parent, The time is 2020-07-24 09:57:32 +0800
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: parent, The time is 2020-07-24 09:58:19 +0800
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: child 1, The time is 2020-07-24 09:58:20 +0800
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: child 0, The time is 2020-07-24 09:58:21 +0800
    Ankitas-MBP:websockets ankitagupta$ ruby client.rb
    Server id: child 1, The time is 2020-07-24 09:58:22 +0800

    Ankitas-MBP:websockets ankitagupta$ lsof -P -i :5678 | grep LISTEN
    ruby    43289 ankitagupta   10u  IPv6 0xd05968e1d897a167      0t0  TCP *:5678 (LISTEN)
    ruby    43316 ankitagupta   10u  IPv6 0xd05968e1d897a167      0t0  TCP *:5678 (LISTEN)
    ruby    43317 ankitagupta   10u  IPv6 0xd05968e1d897a167      0t0  TCP *:5678 (LISTEN)


## Key-takeaways

These implementations were good in aiding my own understanding and appreciating some fundamental Linux concepts.


- Only one process can bind to a single port.
    - In the examples above, I call `server = TCPServer.open(5678)` only once, to bind the main process to port `5678`. Calling this multiple times in your server implementation will cause an error. Try it for yourself!
- Multiple processes can listen to incoming connections on the same port.
    - This is the property that most web servers use today for implementing concurrency across multiple processes.
- Forking a process copies the file descriptors of the parent process to the child process, allowing the child processes to listen to incoming connections on the same port as the parent.
    - In the last implementation calling `Process.fork` is critical to the implementation because it allows the child processes to listen to the same port as the parent process.
- The OS schedules a single process to handle an incoming connection.
    - In the multi-process server case, the OS ends up scheduling a process randomly, and that is the process that ends up handling the incoming connection. This is why the each time I call `ruby client.rb` the connection is randomly handled by one of the children processes or the parent process.
