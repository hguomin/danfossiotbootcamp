# Purpose
Develop an simulated Azure iot hub device

# Steps

## Task 0: Prepare your device app code and environment

1. In you command promot, navigate to the root folder of the PerfSensorDevice C# project
   ```bash
    $ dotnet new console -o PerfSensorDevice
    $ cd PerfSensorDevice
   ```

2. Open source folder in Visual Studio Code, Replace the value of the iothubDeviceConnectionString variable in Program.cs with the device connection string you made a note of earlier. Then save your changes to Program.cs.
   
3. In VS Code, press Ctrl+Shift+` to open the TERMINAL window, then run below command to restore the package for this application
   ```
   $ dotnet restore
   ```
4. Connect to your Azure IoT Hub in VS Code 
   In VS Code, press F1 to open the Command Palette, type "Azure Sign In With Device Code" and select the command to sign in Azure

   ![ ][./images/azure-signin-1.PNG]

   Once you signed in Azure, follow below steps to select you IoT Hub in VS Code

   ![ ][./images/select-iot-hub-1.PNG

   Choose your subscription

   ![][./images/choose-sub.png]

   Choose your IoT Hub

   ![][./images/choose-iot-hub.png]

   Now you will connect to your iot hub in VS Code, you can find you device in the device list under the IoT Hub, as below shows:

   ![][./images/choose-device.png]

## Task 1: Initialize device client
1. Add below code to the below of the line "//TASK-1: create device client from connection string" in MainAsync() method
    ```cs
    deviceClient = DeviceClient.CreateFromConnectionString(iothubDeviceConnectionString);

    ```
## Task 2: Send device telemetry to Azure IoT Hub
1. Add below function code to the below of the comment line "//TASK-2: Send device messgage: SendDeviceTelemetryAsync"
   ```cs
    private static async Task SendDeviceTelemetryAsync()
    {
        while(true)
        {
            //setting sensor data
            var sensorData = new
            {
                CpuUsage = perfCounter.CpuUsage,
                AvailableMemory = perfCounter.MemoryAvailable,
                NetworkLatency = new
                {
                    Host = perfCounter.NetworkHostname,
                    Web = perfCounter.NetworkWebLatency,
                    Ping = perfCounter.NetworkPingLatency
                }
            };

            var msgJson = JsonConvert.SerializeObject(sensorData);
            var message = new Message(Encoding.ASCII.GetBytes(msgJson));

            // Add a custom application property to the message.
            // An IoT hub can filter on these properties without access to the message body.
            message.Properties.Add("cpuLoadAlert", (sensorData.CpuUsage > 80.0) ? "true" : "false");

            // Send the telemetry message
            await deviceClient.SendEventAsync(message);

            Console.WriteLine("\n[DEVICE TELEMETRY]: Sending D2C message: {0} for every {1} ms.", msgJson, telemetryInterval);

            await Task.Delay(telemetryInterval);
        }
    }
   ``
2. Add below code to the below of the comment line "//TASK-2: Send device telemetry "
   ```cs
   tasks.Add(SendDeviceTelemetryAsync());
   ```
3. In command promot issue below command to run the app
   ```bash
   dotnet run
   ```

## Task 3: Handle device command for setting telemetry interval 
1. Add below function code to the below of the comment line "//TASK-3: Handle device command: SetTelemetryInterval"
   ```cs
   private static Task<MethodResponse> SetTelemetryInterval(MethodRequest methodRequest, object context)
   {
       var data = Encoding.UTF8.GetString(methodRequest.Data);
       int dataVal = 0;
       if(Int32.TryParse(data, out dataVal))
       {
           Console.ForegroundColor = ConsoleColor.Red;
           Console.WriteLine("\n[DEVICE METHORD]: Telemetry interval set to {0} ms", dataVal);
           Console.ResetColor();

           lock(telemetryIntervalLock)
           {
               telemetryInterval = dataVal;
           }
           // Acknowlege the direct method call with a 200 success message
           string result = "{\"result\":\"Executed direct method: " + methodRequest.Name + "\"}";
           return Task.FromResult(new MethodResponse(Encoding.UTF8.GetBytes(result), 200));
       }
       else
       {
           // Acknowlege the direct method call with a 400 error message
           string result = "{\"result\":\"Invalid parameter\"}";
           return Task.FromResult(new MethodResponse(Encoding.UTF8.GetBytes(result), 400));
       }
   }

   ``
2. Add below code to the below of the comment line "//TASK-3: register device method for command" in the MainAsync() function to register this device method
   ```cs
   deviceClient.SetMethodHandlerAsync(nameof(SetTelemetryInterval), SetTelemetryInterval, null).Wait();
   ```
3. In command promot issue below command to run the app
   ```bash
   dotnet run
   ```

## Task 4: Handle device command for changging network latency test target
1. Add below function code to the below of the comment line "//TASK-4: Handle device command: SetNetworkLatencyTestTargetHost"
   ```cs
    private static Task<MethodResponse> SetNetworkLatencyTestTargetHost(MethodRequest methodRequest, object context)
    {
       string host = Encoding.UTF8.GetString(methodRequest.Data);
       if(!string.IsNullOrEmpty(host))
       {
           host = host.Trim('"');
           Console.ForegroundColor = ConsoleColor.Yellow;
           Console.WriteLine("\n[DEVICE METHORD]: Network latency test target host set to {0}.", host);
           Console.ResetColor();

           perfCounter.NetworkHostname = host;

           // Acknowlege the direct method call with a 200 success message
           string result = "{\"result\":\"Executed direct method: " + methodRequest.Name + "\"}";
           return Task.FromResult(new MethodResponse(Encoding.UTF8.GetBytes(result), 200));
       }
       else
       {
           // Acknowlege the direct method call with a 400 error message
           string result = "{\"result\":\"Invalid parameter\"}";
           return Task.FromResult(new MethodResponse(Encoding.UTF8.GetBytes(result), 400));
       }
    }
   ```
2. Add below code to the below of the comment line "//TASK-4: register device method for SetNetworkLatencyTestTargetHost" in the MainAsync() function to register this device method
   ```cs
   deviceClient.SetMethodHandlerAsync(nameof(SetNetworkLatencyTestTargetHost), SetNetworkLatencyTestTargetHost, null).Wait();
   ```
3. In command promot issue below command to run the app
   ```bash
   dotnet run
   ```

## Task 5: Receive cloud to device message
1. Add below function code to the below of the comment line "//TASK-5: Receive cloud to device message: ReceiveDeviceMessage"
   ```cs
   private static async Task ReceiveDeviceMessage()
   {
       Console.WriteLine("\n[DEVICE MESSAGE]> Device waiting for commands from IoTHub...");

       Message receivedMessage;
       string messageData;

       while (true)
       {
           receivedMessage = await deviceClient.ReceiveAsync(TimeSpan.FromSeconds(30)).ConfigureAwait(false);

           if (receivedMessage != null)
           {
               messageData = Encoding.ASCII.GetString(receivedMessage.GetBytes());

               Console.ForegroundColor = ConsoleColor.Green;
               Console.WriteLine("\n[DEVICE MESSAGE]> {0}: Received message: {1}", DateTime.Now.ToLocalTime(), messageData);
               Console.ResetColor();

               int propCount = 0;
               foreach (var prop in receivedMessage.Properties)
               {
                   Console.ForegroundColor = ConsoleColor.Green;
                   Console.WriteLine("\n[DEVICE MESSAGE]> Property[{0}> Key={1} : Value={2}", propCount++, prop.Key, prop.Value);
                   Console.ResetColor();
               }

               await deviceClient.CompleteAsync(receivedMessage).ConfigureAwait(false);
           }
           else
           {
               Console.WriteLine("\n[DEVICE MESSAGE]> {0}: Timed out", DateTime.Now.ToLocalTime());
           }
       }
    }
   ```
2. Add below code to the below of the comment line "//TASK-5: receive cloud to device message" in the MainAsync() function to add the receive device message task
   ```cs
   tasks.Add(ReceiveDeviceMessage());
   ```
3. In command promot issue below command to run the app
   ```bash
   dotnet run
   ```