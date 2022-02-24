# content
- [Introduction](#Introduction)
- [generate background](#generate background)
- [How to use](#How to use)
- [TcpSocket](#TcpSocket)
- [WebSocket](#WebSocket)
- [UdpSocket](#UdpSocket)
- [end](#end)

# Introduction
DotNettySocket is a .NET cross-platform Socket framework (supports .NET4.5+ and .NET Standard2.0+), and supports TcpSocket, WebSocket and UdpSocket at the same time. It is based on Microsoft's powerful DotNetty framework and strives to provide the most simple* for Socket communication. *, **efficient**, **elegant** operation.

Installation method: Nuget can install **DotNettySocket**

Project address: https://github.com/Coldairarrow/DotNettySocket

# Background
When I first came into contact with the Internet of Things two years ago, I needed to use Tcp and Udp communication. For the convenience of use, the original Socket was simply encapsulated, which basically met the needs, and the framework was open sourced. However, due to limited energy and strength, the original framework was not further optimized. Later, I discovered the powerful DotNetty framework. DotNetty is a ported version of the open source Java Netty framework of the Microsoft Azure team. It has excellent performance and a strong maintenance team. Many powerful .NET frameworks use it. DotNetty is powerful, but it is not simple enough to use (perhaps it is my personal feeling), just recently the project needs to use WebSocket, so I took the time to simply encapsulate it based on DotNetty, and come up with a simple, efficient and elegant one. Socket frame.

# How to use

## TcpSocket
Tcp is connection-oriented, so it is very important for the server to manage the connection. The framework supports the processing of various events, setting the connection name (identity) for the connection, finding a specific connection through the connection name, connecting to send and receive data, subcontracting, Sticky package handling.
- Server
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Text;
using System.Threading.Tasks;

namespace TcpSocket.Server
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theServer = await SocketBuilderFactory.GetTcpSocketServerBuilder(6001)
                .SetLengthFieldEncoder(2)
                .SetLengthFieldDecoder(ushort.MaxValue, 0, 2, 0, 2)
                .OnConnectionClose((server, connection) =>
                {
                    Console.WriteLine($"Connection closed, connection name [{connection.ConnectionName}], current number of connections: {server.GetConnectionCount()}");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Server exception:{ex.Message}");
                })
                .OnNewConnection((server, connection) =>
                {
                    connection.ConnectionName = $"name{connection.ConnectionId}";
                    Console.WriteLine($"New connection: {connection.ConnectionName}, current number of connections: {server.GetConnectionCount()}");
                })
                .OnRecieve((server, connection, bytes) =>
                {
                    Console.WriteLine($"Server:Data{Encoding.UTF8.GetString(bytes)}");
                    connection.Send(bytes);
                })
                .OnSend((server, connection, bytes) =>
                {
                    Console.WriteLine($"Send data to connection name [{connection.ConnectionName}]:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnServerStarted(server =>
                {
                    Console.WriteLine($"Service Start");
                }).BuildAsync();

            Console.ReadLine();
        }
    }
}
```
- Client
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Net;
using System.Text;
using System.Threading.Tasks;

namespace UdpSocket.Client
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theClient = await SocketBuilderFactory.GetUdpSocketBuilder()
                .OnClose(server =>
                {
                    Console.WriteLine($"Client closes");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Client exception:{ex.Message}");
                })
                .OnRecieve((server, point, bytes) =>
                {
                    Console.WriteLine($"Client:Received data from [{point.ToString()}]:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnSend((server, point, bytes) =>
                {
                    Console.WriteLine($"Client sends data: target[{point.ToString()}] data:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnStarted(server =>
                {
                    Console.WriteLine($"Client Start");
                }).BuildAsync();

            while (true)
            {
                await theClient.Send(Guid.NewGuid().ToString(), new IPEndPoint(IPAddress.Parse("127.0.0.1"), 6003));
                await Task.Delay(1000);
            }
        }
    }
}

```
## WebSocket
The interfaces of WebSocket and TcpSocket are basically the same. The only difference is that TcpSocket supports the sending and receiving of bytes and needs to handle the sub-packet and sticky packets by itself. On the other hand, WebSocket directly transmits and receives string (UTF-8) encoding, and does not need to consider subcontracting and sticking. The framework does not currently support WSS, the recommended solution is to use Nginx forwarding (relevant information can be found by searching)
- Server
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Threading.Tasks;

namespace WebSocket.Server
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theServer = await SocketBuilderFactory.GetWebSocketServerBuilder(6002)
                .OnConnectionClose((server, connection) =>
                {
                    Console.WriteLine($"Connection closed, connection name [{connection.ConnectionName}], current number of connections: {server.GetConnectionCount()}");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Server exception:{ex.Message}");
                })
                .OnNewConnection((server, connection) =>
                {
                    connection.ConnectionName = $"name{connection.ConnectionId}";
                    Console.WriteLine($"New connection: {connection.ConnectionName}, current number of connections: {server.GetConnectionCount()}");
                })
                .OnRecieve((server, connection, msg) =>
                {
                    Console.WriteLine($"Server:Data{msg}");
                    connection.Send(msg);
                })
                .OnSend((server, connection, msg) =>
                {
                    Console.WriteLine($"Send data to connection name [{connection.ConnectionName}]:{msg}");
                })
                .OnServerStarted(server =>
                {
                    Console.WriteLine($"Service Start");
                }).BuildAsync();

            Console.ReadLine();
        }
    }
}

```
- Console client
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Threading.Tasks;

namespace WebSocket.ConsoleClient
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theClient = await SocketBuilderFactory.GetWebSocketClientBuilder("127.0.0.1", 6002)
                .OnClientStarted(client =>
                {
                    Console.WriteLine($"Client Start");
                })
                .OnClientClose(client =>
                {
                    Console.WriteLine($"Client closes");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Exception:{ex.Message}");
                })
                .OnRecieve((client, msg) =>
                {
                    Console.WriteLine($"Client:Received data:{msg}");
                })
                .OnSend((client, msg) =>
                {
                    Console.WriteLine($"Client:Send data:{msg}");
                })
                .BuildAsync();

            while (true)
            {
                await theClient.Send(Guid.NewGuid().ToString());

                await Task.Delay(1000);
            }
        }
    }
}
```
- Web client
``` javascript
<!DOCTYPE HTML>
<html>
<head>
    <meta charset="utf-8">
    <title>Rookie Tutorial (runoob.com)</title>

    <script type="text/javascript">
        function WebSocketTest() {
            if ("WebSocket" in window) {
                var ws = new WebSocket("ws://127.0.0.1:6002");

                ws.onopen = function () {
                    console.log('connect to the server');
                    setInterval(function () {
                        ws.send("111111");
                    }, 1000);
                };

                ws.onmessage = function (evt) {
                    var received_msg = evt.data;
                    console.log('received' + received_msg);
                };

                ws.onclose = function () {
                    console.log("Connection closed...");
                };
            }

            else {
                alert("Your browser does not support WebSocket!");
            }
        }
    </script>

</head>
<body>
    <div id="sse">
        <a href="javascript:WebSocketTest()">Run WebSocket</a>
    </div>
</body>
</html>
```
## UdpSocket
Udp is inherently integrated with sending and receiving. The following is divided into server and client just for the convenience of understanding.
- Server
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Text;
using System.Threading.Tasks;

namespace UdpSocket.Server
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theServer = await SocketBuilderFactory.GetUdpSocketBuilder(6003)
                .OnClose(server =>
                {
                    Console.WriteLine($"Server is closed");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Server exception:{ex.Message}");
                })
                .OnRecieve((server, point, bytes) =>
                {
                    Console.WriteLine($"Server: Received data from [{point.ToString()}]:{Encoding.UTF8.GetString(bytes)}");
                    server.Send(bytes, point);
                })
                .OnSend((server, point, bytes) =>
                {
                    Console.WriteLine($"Server sends data: target[{point.ToString()}]Data:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnStarted(server =>
                {
                    Console.WriteLine($"Server Start");
                }).BuildAsync();

            Console.ReadLine();
        }
    }
}

```
- Client
``` c#
using Coldairarrow.DotNettySocket;
using System;
using System.Net;
using System.Text;
using System.Threading.Tasks;

namespace UdpSocket.Client
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var theClient = await SocketBuilderFactory.GetUdpSocketBuilder()
                .OnClose(server =>
                {
                    Console.WriteLine($"Client closes");
                })
                .OnException(ex =>
                {
                    Console.WriteLine($"Client exception:{ex.Message}");
                })
                .OnRecieve((server, point, bytes) =>
                {
                    Console.WriteLine($"Client:Received data from [{point.ToString()}]:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnSend((server, point, bytes) =>
                {
                    Console.WriteLine($"Client sends data: target[{point.ToString()}] data:{Encoding.UTF8.GetString(bytes)}");
                })
                .OnStarted(server =>
                {
                    Console.WriteLine($"Client Start");
                }).BuildAsync();

            while (true)
            {
                await theClient.Send(Guid.NewGuid().ToString(), new IPEndPoint(IPAddress.Parse("127.0.0.1"), 6003));
                await Task.Delay(1000);
            }
        }
    }
}
```

# end
All the above examples are in the source code. If you think it is good, please like it and add a star. I hope it can help everyone.

If you have any questions, please feedback or join the group to communicate

QQ group 1: (full)

QQ group 2:579202910
