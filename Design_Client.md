# TicTacToe Client (Player 2) Design
> This is the design document for the TicTacToe Client ([tictactoeClient.c](https://github.com/CSE-5462-Spring-2021/assignment-6-conner-ben/blob/main/tictactoeClient.c)).  
> By: Ben Nagel

## Table of Contents
- TicTacToe Class Protocol - [Protocol Document](https://docs.google.com/document/d/1H9yrRi0or_yTt-0xs5QAp2C6umw2qWW1FDL3EyVwF5g/edit?usp=sharing)
- [Environment Constants](#environment-constants)
- [High-Level Architecture](#high-level-architecture)
- [Low-Level Architecturet](#low-level-architecture)

## Environment Constants
```C#
/* Define the number of rows and columns */
#define ROWS 3 
#define COLUMNS 3
/* The number of command line arguments. */
#define NUM_ARGS 3
```

## High-Level Architecture
At a high level, the client application takes in input from the user and trys to send a datagram to the server to start a game. If everything worked, it waits for the server to send the first move of tictactoe via datagram. Once the move was received, the client marks the move on the client board, if the move was valid, otherwise closes the connection. If the move was valid the client sends the server the next move and this processes continues until there is a winner or a tie.
```C
int main(int argc, char *argv[])
{
    struct buffer Buffer = {0};
    char board[ROWS][COLUMNS];
    int sd;
    struct sockaddr_in server_address;
    struct sockaddr_in troll;
    int portNumber;
    char serverIP[29];

    // check for two arguments
    if (argc != 3)
    {
        printf("Wrong number of command line arguments");
        printf("Input is as follows: tictactoeP2 <port-num> <ip-address>");
        exit(1);
    }
    // create the socket
    sd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sd < 0)
    {
        printf("ERROR making the socket");
        exit(1);
    }
    else
    {
        printf("Socket Created\n");
    }

    portNumber = strtol(argv[1], NULL, 10);
    strcpy(serverIP, argv[2]);
    server_address.sin_family = AF_INET;
    server_address.sin_port = htons(portNumber);
    server_address.sin_addr.s_addr = inet_addr(serverIP);

    troll.sin_family = AF_INET;
    troll.sin_port = htons(4444);
    troll.sin_addr.s_addr = INADDR_ANY;
    if (bind(sd, (struct sockaddr *)&troll, sizeof(troll)) < 0)
    {
        printf("ERROR WITH SOCKET\n");
        exit(1);
    }

    // connnect to the sever
    socklen_t fromLength = sizeof(struct sockaddr);
    Buffer.version = 4;
    Buffer.command = 0;
    Buffer.seqNum = 0;
    if (sendto(sd, &Buffer, sizeof(Buffer), 0, (struct sockaddr *)&server_address, fromLength) < 0)
    {
        close(sd);
        perror("error    connecting    stream    socket");
        exit(1);
    }

    printf("Connected to the server!\n");
    initSharedState(board);                                   // Initialize the 'game' board
    tictactoe(board, sd, (struct sockaddr *)&server_address); // call the 'game'
    return 0;
}
```

## Low-Level Architecture
- Client wins and waits for GAME_OVER COMMAND
```C
int c = 0;
                int gameover = 0;
                // Waits for a GAME_OVER datagram from Player 1 
                for (c = 0; c < 4 && gameover == 0; c++)
                {
                    printf("Waiting for player 1 to issue a GAME_OVER command...\n");
                    set_timeout(sd, 30);   // sets timeout
                    rc = recvfrom(sd, &player1, sizeof(player1), 0, (struct sockaddr *)serverAdd, &fromLength); 
                    printf("Player 1 version: %d , SeqNum: %d , Command: %d , Data: %c GameNumber %d \n", player1.version, player1.seqNum, player1.command, player1.data, player1.gameNumber);
                   
                    if (rc <= 0)
                    {
                        // checks if there was a timeout
                        if ((errno == EAGAIN || errno == EWOULDBLOCK))
                        {
                            if (c != 3)
                            {
                                printf("ERROR: TIMEOUT #%d\n", c);
                                printf("Client hasnt gotten a move back from the sever in a while...\n");
                                printf("Resending recent datagram..\n");
                                printf("Resend Version: %d , SeqNumber %d , Command %d, Data %c\n", player2.version, player2.seqNum, player2.command, player2.data);
                                rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);
                                continue;
                            }
                            else
                            {
                                printf("Error: Player ran out of time to respond\n");
                                printf("Closing connection!\n");
                                exit(1);
                            }
                        }
                        printf("Connection lost!\n");
                        printf("Closing connection!\n");
                        exit(1);
                    }
                    else if (player1.command == 1)
                    {
                        printf("ERROR: DIDNT GET GAME_OVER RESENDING BEFORE GAME OVER\n");
                        printf("Resend Version: %d , SeqNumber %d , Command %d, Data %c\n", player2.version, player2.seqNum, player2.command, player2.data);
                        rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);
                    }
                    else if (player1.command == 2)
                    {
                        printf("GAME_OVER Command recevied!\n");
                        gameover = 1;
                    }
                }
            }

```
- Server wins and waits for a GAME_OVER COMMAND from the client
```C
 printf("Player 1 has signaled that the game has ended...\n");
                printf("Player 2 responding that it has got GAME_OVER from player 1...\n");
                player2.seqNum++;
                player2.command = 2;
                int b = 0;
                int gameover = 0;
                // sends GAME_OVER command and resends the GAME_OVER command if needed
                for (b = 0; b != 3 && gameover == 0; b++)
                {
                    printf("GameOverSend Version: %d , SeqNumber %d , Command %d, Data %c\n", player2.version, player2.seqNum, player2.command, player2.data);
                    rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);
                    if (rc < 0)
                    {
                        printf("%d\n", rc);
                        printf("Connection lost!\n");
                        printf("Closing connection!\n");
                        printf("Bye\n");
                        exit(1);
                    }
                    set_timeout(sd, 60);
                    rc = recvfrom(sd, &player1, sizeof(player1), 0, (struct sockaddr *)serverAdd, &fromLength);
                    if (rc <= 0)
                    {
                        if ((errno == EAGAIN || errno == EWOULDBLOCK))
                        {
                            printf("Player 2 assuming that Player 1 got the GAME_OVER Command because of the no response\n");
                            gameover = 1;
                            continue;
                        }
                        
                        printf("Connection lost!\n");
                        printf("Closing connection!\n");
                        exit(1);
                    }
                    else if (player1.seqNum + 1 == player2.seqNum)
                        {
                            printf("Error: Player 1 didn't get GAME_OVER COMMAND\n");
                            printf("Resending GAMEOVER COMMAND\n");
                            continue;
                        }

```
- Client checks for duplicate datagrams and resends. Also timesout if needed and resends the datagram
```C
 printf("Waiting for square selection from player 1..\n"); // gets chosen spot from player 1
            rc = recvfrom(sd, &player1, sizeof(player1), 0, (struct sockaddr *)serverAdd, &fromLength);
            pick = player1.data;
            if (gameNumber == 0)
            {
                gameNumber = player1.gameNumber;
            }
            printf("Player1SeqNum: %d Player2seqNum: %d\n", player1.seqNum, player2.seqNum);

            /*  if (player1.seqNum == player2.seqNum &&player1.seqNum!=0)
            {
                printf("Datagram recived was 1 behind...\n");
                printf("Resending last Datagram...\n");
                rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);
                 continue;
                }
                */

            // checks to see if the connection was cut mid stream
            if (rc <= 0)
            {
                // checks for a timeout
                if ((errno == EAGAIN || errno == EWOULDBLOCK))
                {
                    if (timeout != 3)
                    {
                        // if timed out it resends the previous datagram
                        WrongSeq = 0;
                        printf("ERROR: TIMEOUT #%d\n", timeout);
                        printf("Client hasnt gotten a move back from the sever in a while...\n");
                        printf("Resending recent datagram..\n");
                        if (player1.seqNum == 0)
                        {
                            player2.version = 4;
                            player2.command = 0;
                            player2.seqNum = 0;
                        }

                        printf("Resend Version: %d , SeqNumber %d , Command %d, Data %c\n", player2.version, player2.seqNum, player2.command, player2.data);
                        rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);

                        timeout++;
                        continue;
                    }
                    printf("Error: Player ran out of time to respond\n");
                    printf("Closing connection!\n");
                    exit(1);
                }
                printf("Connection lost!\n");
                printf("Closing connection!\n");
                exit(1);
            }
            // checks to see if a duplicate datagram is recevied 
            // resends previous datagram
            if (player1.seqNum != player2.seqNum + 1 && player1.seqNum != 0)
            {
                printf("Expected Sequence number is wrong...\n");
                printf("Looking for next sequcence number...\n");
                printf("Resend Version: %d , SeqNumber %d , Command %d, Data %c\n", player2.version, player2.seqNum, player2.command, player2.data);
                rc = sendto(sd, &player2, sizeof(player2), 0, (struct sockaddr *)serverAdd, fromLength);
                WrongSeq = 1;
                continue;
            }
            WrongSeq = 0;
            player2.seqNum = player1.seqNum;
            // checks for invalid datagram
            printf("Player version: %d , SeqNum: %d , Command: %d , Data: %c GameNumber %d \n", player1.version, player1.seqNum, player1.command, player1.data, player1.gameNumber);
            printf("gameNumber: %d \n", gameNumber);
            if (player1.command == 0 || player1.version != 4 || gameNumber != player1.gameNumber)
            {
                printf("Player 1 sent invalid datagram\n");
                printf("Closing connection!\n");
                exit(1);
            }
        }
```
