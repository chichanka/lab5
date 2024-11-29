#include <winsock2.h>
#include <stdio.h>
#include <stdlib.h>
#include <process.h>
#include <json-c/json.h> // Бібліотека для роботи з JSON

#pragma comment(lib, "ws2_32.lib")

#define PORT 8888
#define BUFFER_SIZE 1024

void print_file_info(const char *filename) {
    WIN32_FILE_ATTRIBUTE_DATA fileInfo;
    if (GetFileAttributesEx(filename, GetFileExInfoStandard, &fileInfo)) {
        printf("Size: %lld bytes\n", ((long long)fileInfo.nFileSizeHigh << 32) + fileInfo.nFileSizeLow);

  FILETIME ftCreate, ftAccess, ftWrite;
        SYSTEMTIME stUTC, stLocal;

ftCreate = fileInfo.ftCreationTime;
        ftAccess = fileInfo.ftLastAccessTime;
        ftWrite = fileInfo.ftLastWriteTime;

FileTimeToSystemTime(&ftCreate, &stUTC);
        SystemTimeToTzSpecificLocalTime(NULL, &stUTC, &stLocal);
        printf("Created: %02d/%02d/%d %02d:%02d\n",
            stLocal.wMonth, stLocal.wDay, stLocal.wYear, stLocal.wHour, stLocal.wMinute);

FileTimeToSystemTime(&ftAccess, &stUTC);
        SystemTimeToTzSpecificLocalTime(NULL, &stUTC, &stLocal);
        printf("Last Accessed: %02d/%02d/%d %02d:%02d\n",
            stLocal.wMonth, stLocal.wDay, stLocal.wYear, stLocal.wHour, stLocal.wMinute);

FileTimeToSystemTime(&ftWrite, &stUTC);
        SystemTimeToTzSpecificLocalTime(NULL, &stUTC, &stLocal);
        printf("Last Modified: %02d/%02d/%d %02d:%02d\n",
            stLocal.wMonth, stLocal.wDay, stLocal.wYear, stLocal.wHour, stLocal.wMinute);

DWORD dwRtnCode = 0;
        PSID pOwnerSID = NULL;
        PSECURITY_DESCRIPTOR pSD = NULL;

dwRtnCode = GetNamedSecurityInfo(
            filename,
            SE_FILE_OBJECT,
            OWNER_SECURITY_INFORMATION,
            &pOwnerSID,
            NULL,
            NULL,
            NULL,
            &pSD);

if (dwRtnCode == ERROR_SUCCESS) {
            char ownerName[256];
            char domainName[256];
            DWORD ownerNameSize = sizeof(ownerName);
            DWORD domainNameSize = sizeof(domainName);
            SID_NAME_USE sidType;
            if (LookupAccountSid(NULL, pOwnerSID, ownerName, &ownerNameSize, domainName, &domainNameSize, &sidType)) {
                printf("Owner: %s\\%s\n", domainName, ownerName);
            } else {
                printf("LookupAccountSid error: %lu\n", GetLastError());
            }
            LocalFree(pSD);
        } else {
            printf("GetNamedSecurityInfo error: %lu\n", dwRtnCode);
        }
    } else {
        printf("GetFileAttributesEx error: %lu\n", GetLastError());
    }
}

void client_handler(void *socket_desc) {
    SOCKET s = *(SOCKET*)socket_desc;
    char buffer[BUFFER_SIZE];
    int recv_size;

if((recv_size = recv(s, buffer, BUFFER_SIZE, 0)) == SOCKET_ERROR) {
        printf("recv failed: %d\n", WSAGetLastError());
        closesocket(s);
        return;
    }
    buffer[recv_size] = '\0';

printf("Received from client: %s\n", buffer);

char *message = "Hello Client, I have received your connection. But I have to go now, bye\n";
    send(s, message, strlen(message), 0);

closesocket(s);
    _endthread();
}

void tcp_server() {
    WSADATA wsa;
    SOCKET s, new_socket;
    struct sockaddr_in server, client;
    int c;
    HANDLE thread;

printf("Initialising Winsock...\n");
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0) {
        printf("Failed. Error Code: %d\n", WSAGetLastError());
        return;
    }
    printf("Initialised.\n");

if ((s = socket(AF_INET, SOCK_STREAM, 0)) == INVALID_SOCKET) {
        printf("Could not create socket: %d\n", WSAGetLastError());
        return;
    }
    printf("Socket created.\n");

server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

if (bind(s, (struct sockaddr *)&server, sizeof(server)) == SOCKET_ERROR) {
        printf("Bind failed with error code: %d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return;
    }
    printf("Bind done.\n");

listen(s, 3);

printf("Waiting for incoming connections...\n");
    c = sizeof(struct sockaddr_in);
    while ((new_socket = accept(s, (struct sockaddr *)&client, &c)) != INVALID_SOCKET) {
        printf("Connection accepted.\n");
        thread = (HANDLE)_beginthread(client_handler, 0, (void*)&new_socket);
    }

if (new_socket == INVALID_SOCKET) {
        printf("Accept failed with error code: %d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return;
    }

closesocket(s);
    WSACleanup();
}

void tcp_client(const char *server_ip) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in server;
    char *message, server_reply[BUFFER_SIZE];
    int recv_size;

  printf("Initialising Winsock...\n");
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0) {
        printf("Failed. Error Code: %d\n", WSAGetLastError());
                
return;
                
}
    printf("Initialised.\n");

if ((s = socket(AF_INET, SOCK_STREAM, 0)) == INVALID_SOCKET) {
        printf("Could not create socket: %d\n", WSAGetLastError());
        return;
    }
    printf("Socket created.\n");

server.sin_addr.s_addr = inet_addr(server_ip);
    server.sin_family = AF_INET;
    server.sin_port = htons(PORT);

if (connect(s, (struct sockaddr *)&server, sizeof(server)) < 0) {
        printf("Connect error: %d\n", WSAGetLastError());
        return;
    }
    printf("Connected.\n");

   message = "Hello Server!";
    if (send(s, message, strlen(message), 0) < 0) {
        printf("Send failed: %d\n", WSAGetLastError());
        return;
    }
    printf("Data sent.\n");

if ((recv_size = recv(s, server_reply, BUFFER_SIZE, 0)) == SOCKET_ERROR) {
        printf("recv failed: %d\n", WSAGetLastError());
    }
    server_reply[recv_size] = '\0';
    printf("Server reply: %s\n", server_reply);

  closesocket(s);
    WSACleanup();
}

void udp_server() {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in server, client;
    char buffer[BUFFER_SIZE];
    int slen, recv_len;
    slen = sizeof(client);

  printf("Initialising Winsock...\n");
    if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0) {
        printf("Failed. Error Code: %d\n", WSAGetLastError());
        return;
    }
    printf("Initialised.\n");

  if ((s = socket(AF_INET, SOCK_DGRAM, 0)) == INVALID_SOCKET) {
        printf("Could not create socket: %d\n", WSAGetLastError());
        return;
    }
    printf("Socket created.\n");

server.sin_family = AF_INET;
    server.sin_addr.s_addr = INADDR_ANY;
    server.sin_port = htons(PORT);

if (bind(s, (struct sockaddr *)&server, sizeof(server)) == SOCKET_ERROR) {
        printf("Bind failed with error code: %d\n", WSAGetLastError());
        closesocket(s);
        WSACleanup();
        return;
    }
    printf("Bind done.\n");

while (1) {
        printf("Waiting for data...\n");
        fflush(stdout);

if ((recv_len = recvfrom(s, buffer, BUFFER_SIZE, 0, (struct sockaddr *)&client, &slen)) == SOCKET_ERROR) {
            printf("recvfrom() failed with error code: %d\n", WSAGetLastError());
            closesocket(s);
            WSACleanup();
            return;
        }

buffer[recv_len] = '\0';
        printf("Received packet from %s:%d\n", inet_ntoa(client.sin_addr), ntohs(client.sin_port));
        printf("Data: %s\n", buffer);

sendto(s, buffer, recv_len, 0, (struct sockaddr *)&client, slen);
    }

closesocket(s);
    WSACleanup();
}

void udp_client(const char *server_ip) {
    WSADATA wsa;
    SOCKET s;
    struct sockaddr_in server;
    char message[BUFFER_SIZE], server_reply[BUFFER_SIZE];
    int slen = sizeof(server), recv_len;

