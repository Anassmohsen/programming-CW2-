// server code

#include <iostream>
#include <vector>
#include <thread>
#include <mutex>
#include <algorithm>
#include <winsock2.h>
#include <ws2tcpip.h>

using namespace std;

#pragma comment(lib, "ws2_32.lib")  // Link against the Windows Socket library

mutex clientsMutex;                // Mutex to protect access to the clientSockets vector
vector<SOCKET> clientSockets;      // Vector to keep track of connected client sockets

// Initializes Winsock, necessary for any network communication on Windows
bool InitializeWinsock() {
    WSADATA data;
    int result = WSAStartup(MAKEWORD(2, 2), &data);
    if (result != 0) {
        cerr << "Winsock initialization failed with error: " << result << endl;
        return false;
    }
    return true;
}

// Broadcasts a message to all clients except the sender
void BroadcastMessage(const string& message, SOCKET senderSocket) {
    lock_guard<mutex> lock(clientsMutex); // Ensure thread-safe access to the clientSockets
    for (auto& sock : clientSockets) {
        if (sock != senderSocket) {  // Avoid sending the message back to the sender
            int sendResult = send(sock, message.c_str(), message.length(), 0);
            if (sendResult == SOCKET_ERROR) {
                cerr << "Failed to send message to client, error: " << WSAGetLastError() << endl;
            }
        }
    }
}

// Handles incoming data from a connected client
void HandleClient(SOCKET clientSocket) {
    {
        lock_guard<mutex> lock(clientsMutex);
        clientSockets.push_back(clientSocket); // Add the new client to the list of clients
    }

    char buffer[4096]; // Buffer to store data received from the client
    try {
        while (true) {
            memset(buffer, 0, sizeof(buffer));
            int bytesReceived = recv(clientSocket, buffer, sizeof(buffer), 0);
            if (bytesReceived <= 0) {
                if (bytesReceived == 0) {
                    cout << "Client disconnected gracefully." << endl;
                }
                else {
                    cerr << "Receive error: " << WSAGetLastError() << endl;
                }
                break;
            }

            string message(buffer, bytesReceived);
            cout << "Received: " << message << " from client." << endl;
            BroadcastMessage(message, clientSocket); // Broadcast the received message to other clients
        }
    }
    catch (const exception& e) {
        cerr << "Exception in handleClient: " << e.what() << endl;
    }

    // Cleanup after client disconnection
    {
        lock_guard<mutex> lock(clientsMutex);
        clientSockets.erase(remove(clientSockets.begin(), clientSockets.end(), clientSocket), clientSockets.end()); // Remove the client from the list
    }
    closesocket(clientSocket); // Close the client socket
    cout << "Client socket closed." << endl;
}

// Main function to set up the server and accept incoming client connections
int main() {
    if (!InitializeWinsock()) {
        return 1;
    }

    SOCKET listenSocket = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (listenSocket == INVALID_SOCKET) {
        cerr << "Socket creation failed: " << WSAGetLastError() << endl;
        WSACleanup();
        return 1;
    }

    sockaddr_in serverAddr;
    serverAddr.sin_family = AF_INET;
    serverAddr.sin_port = htons(12345);
    serverAddr.sin_addr.s_addr = INADDR_ANY;

    if (bind(listenSocket, (sockaddr*)&serverAddr, sizeof(serverAddr)) == SOCKET_ERROR) {
        cerr << "Bind failed: " << WSAGetLastError() << endl;
        closesocket(listenSocket);
        WSACleanup();
        return 1;
    }

    if (listen(listenSocket, SOMAXCONN) == SOCKET_ERROR) {
        cerr << "Listen failed: " << WSAGetLastError() << endl;
        closesocket(listenSocket);
        WSACleanup();
        return 1;
    }

    cout << "Server is listening on port 12345..." << endl;
    while (true) {
        SOCKET clientSocket = accept(listenSocket, NULL, NULL);
        if (clientSocket != INVALID_SOCKET) {
            thread(HandleClient, clientSocket).detach();  // Start a new thread for each connected client
        }
        else {
            cerr << "Failed to accept client: " << WSAGetLastError() << endl;
        }
    }

    // Cleanup (theoretically unreachable)
    closesocket(listenSocket);
    WSACleanup();
    return 0;
}



// client code

#include <iostream>
#include <string>
#include <thread>
#include <WinSock2.h>
#include <WS2tcpip.h>

using namespace std;

#pragma comment(lib, "ws2_32.lib") // Link the Winsock Library for network communication

// Initialize WinSock for network communication
bool Initialize() {
    WSADATA data;
    // Start Winsock with version 2.2; necessary before using any Winsock functions
    if (WSAStartup(MAKEWORD(2, 2), &data) != 0) {
        cerr << "Winsock initialization failed." << endl;
        return false;
    }
    return true;
}

// Caesar Cipher encryption function
// Encrypts only alphabetic characters and preserves other characters
string Encrypt(const string& text, int shift) {
    string encrypted = text;
    for (char& c : encrypted) {
        if (isalpha(c)) { // Encrypt only alphabetic characters
            char base = islower(c) ? 'a' : 'A';
            c = static_cast<char>((c - base + shift) % 26 + base); // Shift character within the alphabet
        }
    }
    return encrypted;
}

// Caesar Cipher decryption function
// Decrypts a message by reversing the encryption shift
string Decrypt(const string& text, int shift) {
    return Encrypt(text, 26 - shift); // Decrypt by shifting in the opposite direction
}

// Thread function to continuously receive messages from the server
// It decrypts each message received and displays it to the user
void ReceiveMessages(SOCKET sock) {
    char buffer[4096]; // Buffer to hold incoming data
    while (true) {
        memset(buffer, 0, sizeof(buffer)); // Clear the buffer
        int bytesReceived = recv(sock, buffer, sizeof(buffer), 0); // Block until data is received
        if (bytesReceived > 0) {
            // Output decrypted messages from the server
            cout << "\n" << Decrypt(string(buffer, 0, bytesReceived), 5) << "\n> ";
        }
        else {
            cout << "Connection closed by server." << endl;
            break; // Exit loop if the connection is closed or an error occurs
        }
    }
}

int main() {
    if (!Initialize()) { // Ensure Winsock is initialized
        return 1;
    }

    SOCKET s = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP); // Create a TCP socket
    if (s == INVALID_SOCKET) {
        cerr << "Socket creation failed." << endl;
        WSACleanup();
        return 1;
    }

    sockaddr_in serverAddr; // Server address structure
    serverAddr.sin_family = AF_INET; // Set the family to IPv4
    serverAddr.sin_port = htons(12345); // Port number
    inet_pton(AF_INET, "127.0.0.1", &serverAddr.sin_addr); // Convert IP address from text to binary form

    // Attempt to connect to the server
    if (connect(s, reinterpret_cast<sockaddr*>(&serverAddr), sizeof(serverAddr)) == SOCKET_ERROR) {
        cerr << "Connection failed: " << WSAGetLastError() << endl;
        closesocket(s);
        WSACleanup();
        return 1;
    }

    cout << "Connected to the server.\n";
    cout << "Enter your username: ";
    string username;
    getline(cin, username);
    username += ": "; // Append a colon and space after the username for message formatting

    // Start a thread to handle incoming messages
    thread receiveThread(ReceiveMessages, s);

    // Main thread handles user input and sending messages
    cout << "> ";
    string userInput;
    while (getline(cin, userInput)) {
        if (!userInput.empty()) {
            string message = username + userInput; // Prepend username to the message
            string encryptedMessage = Encrypt(message, 5); // Encrypt the message before sending
            if (send(s, encryptedMessage.c_str(), encryptedMessage.length(), 0) == SOCKET_ERROR) {
                cerr << "Failed to send message: " << WSAGetLastError() << endl;
                break;
            }
        }
        cout << "> ";
    }

    receiveThread.join(); // Wait for the receive thread to complete
    closesocket(s); // Close the socket
    WSACleanup(); // Clean up Winsock
    return 0;
}
