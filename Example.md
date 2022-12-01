# Example

## 使用HyperDbg实时记录寄存器内容

- 客户机负责发送的代码

  该代码负责接受HyperDbg向指定Pipe发送的数据，然后将其打印出来

```
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Pipe address : \\\\.\\pipe\\HyperDbgTestPipe \nListening...\n");
            var server = new NamedPipeServerStream("HyperDbgTestPipe");
            server.WaitForConnection();
            StreamReader reader = new StreamReader(server);
            while (true)
            {
                var line = reader.ReadLine();
                Console.WriteLine(line);
            }

        }
    }
```



- 主机负责接受的代码

  该代码负责接受客户机的打印数据，从而监控客户机特定寄存器的内容

  ```
  class MyTcpListener
  {
      public static void Main()
      {
          TcpListener server = null;
          try
          {
              //
              // Set the TcpListener on port 50000.
              //
              Int32 port = 50000;
  
              //
              // Local Ip address
              //
              IPAddress localAddr = IPAddress.Parse("192.168.138.1");
  
              //
              // TcpListener server = new TcpListener(port);
              //
              server = new TcpListener(localAddr, port);
  
              //
              // Start listening for client requests.
              //
              server.Start();
  
              //
              // Buffer for reading data
              //
              Byte[] bytes = new Byte[10000];
              String data = null;
  
              //
              // Enter the listening loop.
              //
              while (true)
              {
                  Console.WriteLine("Waiting for a connection...");
  
                  //
                  // Perform a blocking call to accept requests.
                  // You could also use server.AcceptSocket() here.
                  //
                  TcpClient client = server.AcceptTcpClient();
                  Console.WriteLine("Connected!");
  
                  data = null;
  
                  //
                  // Get a stream object for reading and writing
                  //
                  NetworkStream stream = client.GetStream();
  
                  int i;
  
                  //
                  // Loop to receive all the data sent by the client.
                  //
                  while ((i = stream.Read(bytes, 0, bytes.Length)) != 0)
                  {
                      //
                      // Translate data bytes to a ASCII string.
                      //
                      data = System.Text.Encoding.ASCII.GetString(bytes, 0, i);
                      Console.Write("{0}", data);
  
                  }
  
                  //
                  // Shutdown and end connection
                  //
                  client.Close();
              }
          }
          catch (SocketException e)
          {
              Console.WriteLine("SocketException: {0}", e);
          }
          finally
          {
              //
              // Stop listening for new clients.
              //
              server.Stop();
          }
  
          Console.WriteLine("\nHit enter to continue...");
          Console.Read();
      }
  }
  
  ```

  

- 使用HyperBdg软件对特定CPU寄存器进行监控，并输出给主机，或输出为一个文本文档，或输出到一个特定的pipe

  

  创建输出方式：

  ```
  output create MyOutputName1 file c:\users\sina\desktop\output.txt //输出到一个文本文件
  output create MyOutputName2 tcp 192.168.1.10:8080 //输出到一个指定的ip地址
  output create MyOutputName3 namedpipe \\.\Pipe\HyperDbgOutput //输出到一个指定的管道
  ```

  使用 `output` 命令可以查看各个输出方式的状态，创建的输出方式默认为not open,需要使用`output open MyOutputName`来开启
  然后就可以使用脚本来进行输出

  例如：

  ```
  !syscall script { print(@rax); } output {MyOutputName1,MyOutputName2,MyOutputName3}
  ```

  

