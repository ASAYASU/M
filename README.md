using System;
using System.Net;
using System.Net.Sockets;
using System.Threading;
using System.Collections.Concurrent; //ConcurrentBag<T>型など
using Forms = System.Windows.Forms; //Timer型がSystem.Collections.Concurrentライブラリと被るので、Formsにエイリアス化

namespace WindowsFormsServerApp
{
    public partial class ModbusTCPServer : Forms.Form
    {
        private TcpListener _tcpListener; //502ポート監視用の非公開メンバ変数
        private TcpListener _dashboardListener; //
        private Forms.Timer _uiUpdateTimer;
        
        private ushort[] _holdingRegisters = new ushort[10]; //保持レジスタ配列インスタンス生成
        private ushort[] _inputRegisters = new ushort[20]; //入力レジスタ配列インスタンス生成
        private Random _random = new Random();
        private ConcurrentBag<TcpClient> _dashboardClients = new ConcurrentBag<TcpClient>(); //クライアントの書き込み要求をConcurrentBagで複数スレッドからの処理を同時に行える（スレッドセーフ）状態にする

        public ModbusTCPServer()
        {
            InitializeComponent();
            SetupModbusTCPServer();
            SetupUiTimer();
        }

        public SetupModbusTCPServer()
        {
            try
            {
                ushort port = 502; //Modbus標準のポート番号

                _tcpListener = new TcpListener(IPAddress.Loopback, port); //502ポートをループバックアドレスに開放するインスタンス生成（サーバー側はIPAddress.Loopbackで127.0.0.1を指定する必要あり）
                _tcpListener.Start(); //502ポート開放

                _dashboardListener.Start();

                Thread clientThread = new Thread(HandleClients);
                clientThread.Start();

                _dashboardListener = new TcpListener(IPAddress.Any, dashboardPort);

                //初期データ投入
                for (int i = 0; i < 10; i++)
                {
                    _holdingRegisters[i] = (ushort)(100 + i + 1);
                }
                for (int i = 0; i < 20; i++)
                {
                    _inputRegisters[i] = (ushort)(200 + i + 1);
                }

                lblStatus.Text = "サーバー稼働中";
            }

            //例外処理
            catch (Exception ex) //exにエラーの内容を受ける
            {
                Forms.MessageBox.Show($"サーバー起動エラー");
            }
        }

        

        public void Start()
        {
            

            

            Thread dashboardThread = new Thread(HandleDashboardClients);
            dashboardThread.Start();

            Thread simulationThread = new Thread(SimulateConveyorBelt);
            simulationThread.Start();

            while (_isRunning)
            {
                Thread.Sleep(100);
            }
        }

        private void HandleClients()
        {
            while (_isRunning)
            {
                TcpClient client = _tcpListener.AcceptTcpClient();
                Thread clientThread = new Thread(new ParameterizedThreadStart(HandleClient));
                clientThread.Start(client);
            }
        }

        private void HandleClient(object obj)
        {
            TcpClient tcpClient = (TcpClient)obj;
            NetworkStream stream = tcpClient.GetStream();

            byte[] buffer = new byte[1024];
            int bytesRead;

            while ((bytesRead = stream.Read(buffer, 0, buffer.Length)) != 0)
            {
                byte[] response = ProcessModbusRequest(buffer, bytesRead);
                stream.Write(response, 0, response.Length);
            }

            tcpClient.Close();
        }

        private void HandleDashboardClients()
        {
            while (_isRunning)
            {
                TcpClient dashboardClient = _dashboardListener.AcceptTcpClient(); //クライアントからのダッシュボード書き込みの接続要求を受け入れ
                _dashboardClients.Add(dashboardClient); //
                Console.WriteLine("New dashboard client connected");
            }
        }

        private void BroadcastToDashboards()
        {
            string data = string.Format("{0},{1},{2},{3}",
                _conveyorRunning,
                _conveyorSpeed,
                _itemCount,
                _emergencyStop);
            byte[] message = System.Text.Encoding.ASCII.GetBytes(data);

            foreach (var client in _dashboardClients)
            {
                try
                {
                    NetworkStream stream = client.GetStream();
                    stream.Write(message, 0, message.Length);
                }
                catch
                {
                    // Remove disconnected clients
                    TcpClient removedClient;
                    _dashboardClients.TryTake(out removedClient);
                }
            }
        }

        public void SimulateConveyorBelt()
        {
            while (_isRunning)
            {
                if (_conveyorRunning && !_emergencyStop)
                {
                    // Simulate item movement
                    if (_random.Next(100) < _conveyorSpeed)
                    {
                        _itemCount++;
                        _holdingRegisters[2] = _itemCount;
                        Console.WriteLine("Item passed through. Total count: " + _itemCount);
                    }
                }

                BroadcastToDashboards();
                Thread.Sleep(100); // Update every 100ms
            }
        }

        private byte[] ProcessModbusRequest(byte[] request, int length)
        {
            ushort transactionId = (ushort)((request[0] << 8) | request[1]);
            ushort protocolId = (ushort)((request[2] << 8) | request[3]);
            ushort dataLength = (ushort)((request[4] << 8) | request[5]);
            byte unitId = request[6];
            byte functionCode = request[7];

            Console.WriteLine("Received request - Function: {0}, Transaction ID: {1}", functionCode, transactionId);

            switch (functionCode)
            {
                case 3: // Read Holding Registers
                    return HandleReadHoldingRegisters(request, transactionId, unitId);
                case 6: // Write Single Register
                    return HandleWriteSingleRegister(request, transactionId, unitId);
                default:
                    return CreateExceptionResponse(transactionId, unitId, functionCode, 1); // Illegal function
            }
        }

        private byte[] HandleReadHoldingRegisters(byte[] request, ushort transactionId, byte unitId)
        {
            ushort startAddress = (ushort)((request[8] << 8) | request[9]);
            ushort quantity = (ushort)((request[10] << 8) | request[11]);

            if (startAddress + quantity > _holdingRegisters.Length)
            {
                return CreateExceptionResponse(transactionId, unitId, 3, 2); // Illegal data address
            }

            byte[] responseData = new byte[quantity * 2];
            for (int i = 0; i < quantity; i++)
            {
                responseData[i * 2] = (byte)(_holdingRegisters[startAddress + i] >> 8);
                responseData[i * 2 + 1] = (byte)(_holdingRegisters[startAddress + i] & 0xFF);
            }

            byte[] response = new byte[9 + responseData.Length];
            response[0] = (byte)(transactionId >> 8);
            response[1] = (byte)(transactionId & 0xFF);
            response[2] = 0; // Protocol ID
            response[3] = 0;
            response[4] = (byte)((responseData.Length + 3) >> 8);
            response[5] = (byte)((responseData.Length + 3) & 0xFF);
            response[6] = unitId;
            response[7] = 3; // Function code
            response[8] = (byte)(responseData.Length);
            Array.Copy(responseData, 0, response, 9, responseData.Length);

            Console.WriteLine("Sent response - Read Holding Registers: Start: {0}, Quantity: {1}", startAddress, quantity);
            return response;
        }

        private byte[] HandleWriteSingleRegister(byte[] request, ushort transactionId, byte unitId)
        {
            ushort address = (ushort)((request[8] << 8) | request[9]);
            ushort value = (ushort)((request[10] << 8) | request[11]);

            if (address >= _holdingRegisters.Length)
            {
                return CreateExceptionResponse(transactionId, unitId, 6, 2); // Illegal data address
            }

            _holdingRegisters[address] = value;

            // Process the write based on the register
            switch (address)
            {
                case 0: // Conveyor Status
                    _conveyorRunning = value != 0;
                    Console.WriteLine("Conveyor is now " + (_conveyorRunning ? "running" : "stopped"));
                    break;
                case 1: // Conveyor Speed
                    _conveyorSpeed = value;
                    Console.WriteLine("Conveyor speed set to " + _conveyorSpeed + "%");
                    break;
                case 3: // Emergency Stop
                    _emergencyStop = value != 0;
                    if (_emergencyStop)
                    {
                        _conveyorRunning = false;
                        _conveyorSpeed = 0;
                        _holdingRegisters[0] = 0; // Update conveyor status
                        _holdingRegisters[1] = 0; // Update conveyor speed
                        Console.WriteLine("Emergency stop activated!");
                    }
                    else
                    {
                        Console.WriteLine("Emergency stop deactivated");
                    }
                    break;
            }

            byte[] response = new byte[12];
            Array.Copy(request, response, 12); // Echo the request for a write response

            Console.WriteLine("Wrote value {0} to register {1}", value, address);
            return response;
        }

        private byte[] CreateExceptionResponse(ushort transactionId, byte unitId, byte functionCode, byte exceptionCode)
        {
            byte[] response = new byte[9];
            response[0] = (byte)(transactionId >> 8);
            response[1] = (byte)(transactionId & 0xFF);
            response[2] = 0; // Protocol ID
            response[3] = 0;
            response[4] = 0;
            response[5] = 3;
            response[6] = unitId;
            response[7] = (byte)(functionCode | 0x80); // Error response
            response[8] = exceptionCode;

            Console.WriteLine("Sent exception response - Function: {0}, Exception: {1}", functionCode, exceptionCode);
            return response;
        }

        

        private void ModbusTCPServer_Load(object sender, EventArgs e)
        {

        }
    }
}
