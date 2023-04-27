#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <dirent.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <errno.h>
#include <time.h>
#include <fcntl.h>
#include <netdb.h>
#include <sys/sendfile.h>
#include <sys/wait.h>
#include <limits.h>

#define PORT 8000
#define MAX_COMMAND 1024
#define MAX_PENDING 5
#define MAX_BUFFER 1024
#define MAX_PATH 4096
#define MAX_FILENAME 256

#define BACKLOG 10
#define MAX_PATH 4096
#define MAX_FILENAME 256
#define BUFFER_SIZE 1024
void tar_directory(const char *directory, const char *output_file, int size1, int size2);
void add_file_to_tar(const char *filename, const char *tar_file);
void send_file(int client_socket, char *filename);
void tar_directory_by_date(char *dir_path, char *output_file, char *date1, char *date2);
int tar_directory_with_files(char *dir, char *output_file, char **filenames, int num_filenames)
{
   // Generate a tar file containing the specified directory and files.
    int ret = 0;
    char command[MAX_COMMAND];
    snprintf(command, MAX_COMMAND, "tar -czf %s -C %s ", output_file, dir);
    for (int i = 0; i < num_filenames; i++)
    {
        strcat(command, filenames[i]);
        strcat(command, " ");
    }
    ret = system(command);
    return ret;
}
// Searches temp.tar.gz file from root directory of server
void search_temp_file(int client_socket, const char *filename, const char *dir_path)
{
    DIR *dir;
    struct dirent *entry;
    struct stat filestat;
    char path[1024];
    int found = 0;
    if ((dir = opendir(dir_path)) == NULL)
    {
        perror("opendir error");
        exit(1);
    }
    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }
        snprintf(path, sizeof(path), "%s/%s", dir_path, entry->d_name);
        if (lstat(path, &filestat) < 0)
        {
            perror("lstat error");
            exit(1);
        }
        if (S_ISDIR(filestat.st_mode))
        {
            search_file(client_socket, filename, path);
        }
        else if (S_ISREG(filestat.st_mode))
        {
            if (strcmp(entry->d_name, filename) == 0)
            {
                char file_details[1024];
                sprintf(file_details, "%ld",
                        filestat.st_size);
                printf("\nPrinting Size: %ld\n", filestat.st_size);
                send(client_socket, file_details, strlen(file_details), 0);
                found = 1;
                break;
            }
        }
    }
    if (!found && strcmp(dir_path, getenv("HOME")) == 0)
    {
        char not_found[1024];
        sprintf(not_found, "File not found\n");
        send(client_socket, not_found, strlen(not_found), 0);
    }
    closedir(dir);
}
void check_tar_file(int client_socket, const char *filename)
{
    search_temp_file(client_socket, filename, getenv("HOME"));
}
// Check validation of filename
void check_file(int client_socket, const char *filename)
{
    search_file(client_socket, filename, getenv("HOME"));
}
void search_file(int client_socket, const char *filename, const char *dir_path)
{
    DIR *dir;
    struct dirent *entry;
    struct stat filestat;
    char path[1024];
    int found = 0;
    if ((dir = opendir(dir_path)) == NULL)
    {
        perror("opendir error");
        exit(1);
    }
    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }
        snprintf(path, sizeof(path), "%s/%s", dir_path, entry->d_name);
        if (lstat(path, &filestat) < 0)
        {
            perror("lstat error");
            exit(1);
        }
        if (S_ISDIR(filestat.st_mode))
        {
            search_file(client_socket, filename, path);
        }
        else if (S_ISREG(filestat.st_mode))
        {
            if (strcmp(entry->d_name, filename) == 0)
            {
                char file_details[1024];
                sprintf(file_details, "Filename: %s\nSize: %ld bytes\nDate created: %s",
                        filename, filestat.st_size, ctime(&filestat.st_mtime));
                send(client_socket, file_details, strlen(file_details), 0);
                found = 1;
                break;
            }
        }
    }
    if (!found && strcmp(dir_path, getenv("HOME")) == 0)
    {
        char not_found[1024];
        sprintf(not_found, "File not found\n");
        send(client_socket, not_found, strlen(not_found), 0);
    }
    closedir(dir);
}
int match_extension(char *filename, char **extensions, int num_extensions)
{
    char *ext = strrchr(filename, '.');
    if (ext == NULL)
    {
        return 0;
    }

    ext++; // skip the dot
    for (int i = 0; i < num_extensions; i++)
    {
        if (strcmp(ext, extensions[i]) == 0)
        {
            return 1;
        }
    }

    return 0;
}

void processclient(int client_socket)
{
    char buffer[MAX_BUFFER];
    int bytes_received;
    int size1, size2;
    while (1)
    {
        // Await a command from the client
        ////////=============Findfile START=============================================================//////////////////

        bytes_received = recv(client_socket, buffer, MAX_BUFFER, 0);
        if (bytes_received == -1)
        {
            perror("recv");
            exit(1);
        }
        // string terminate
        buffer[bytes_received] = '\0';
        // quit command check
        if (strcmp(buffer, "quit") == 0)
        {
            close(client_socket);
            exit(0);
        }
        // findfile command check
        if (strncmp(buffer, "findfile", 8) == 0)
        {
            // extract filename from command
            // "findfile" skip file
            char *filename = buffer + 9; 
            check_file(client_socket, filename);
        }
        ////////=============Findfile END=============================================================//////////////////

        ////////=============sgetfiles START=============================================================//////////////////

        else if (strncmp(buffer, "sgetfiles", 9) == 0)
        {
            char *arg1 = strtok(buffer + 9, " ");
            char *arg2 = strtok(NULL, " ");

            size1 = atoi(arg1);
            size2 = atoi(arg2);

            char output_file[MAX_FILENAME];
            snprintf(output_file, MAX_FILENAME, "temp.tar.gz");

            tar_directory(getenv("HOME"), output_file, size1, size2);

            printf("Output file created: %s\n", output_file);
            char *filename = "temp.tar.gz";
            check_tar_file(client_socket, filename);
            send_file(client_socket, output_file);
            char ackbuffer[1024] = {0};
            int ackread = read(client_socket, ackbuffer, 1024);
            printf("%s\n", ackbuffer);
            if (strncmp(ackbuffer, "Success", 7) == 0)
            {
                char *temp = "temp.tar.gz";

                // remove() function to delete file
                if (remove(temp) == 0)
                {
                    printf("File %s deleted successfully\n", temp);
                }
                else
                {
                    fprintf(stderr, "Error: unable to delete file %s\n", temp);
                }
            }
        }
        ////////=============sgetfiles END=============================================================//////////////////

        ////////=============dgetfiles START=============================================================//////////////////

        else if (strncmp(buffer, "dgetfiles", 9) == 0)
        {
            char *date1 = strtok(buffer + 9, " ");
            char *date2 = strtok(NULL, " ");
            // Extract date1, date2, and -u option

            // Generate a tar file that includes files created between date1 and date2.
            char output_file[MAX_FILENAME];
            snprintf(output_file, MAX_FILENAME, "temp.tar.gz");
            tar_directory_by_date(getenv("HOME"), output_file, date1, date2);
            printf("Output file created: %s\n", output_file);
            char *filename = "temp.tar.gz";
            check_tar_file(client_socket, filename);
            send_file(client_socket, output_file);

            // remove() function to delete file
            char ackbuffer[1024] = {0};
            int ackread = read(client_socket, ackbuffer, 1024);

            printf("%s\n", ackbuffer);
            if (strncmp(ackbuffer, "Success2", 8) == 0)
            {
                char *temp = "temp.tar.gz";

                // remove() function to delete file
                if (remove(temp) == 0)
                {
                    printf("File %s deleted successfully\n", temp);
                }
                else
                {
                    fprintf(stderr, "Error: unable to delete file %s\n", temp);
                }
            }
        }
        ////////=============dgetfiles END=============================================================//////////////////

        ////////=============gettargz START=============================================================//////////////////
        // else if (strncmp(buffer, "gettargz", 8) == 0)
        // {
        //     char *arg1 = strtok(buffer + 8, " ");
        //     char *arg2 = strtok(NULL, " ");
        //     char *arg3 = strtok(NULL, " ");
        //     char *arg4 = strtok(NULL, " ");
        //     char *arg5 = strtok(NULL, " ");
        //     char *arg6 = strtok(NULL, " ");

        //     int extension_count = 0;
        //     char extension_list[6][10];

        //     // Store up to 6 different file types in the extension_list
        //     if (arg1 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg1, 10);
        //         extension_count++;
        //     }
        //     if (arg2 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg2, 10);
        //         extension_count++;
        //     }
        //     if (arg3 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg3, 10);
        //         extension_count++;
        //     }
        //     if (arg4 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg4, 10);
        //         extension_count++;
        //     }
        //     if (arg5 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg5, 10);
        //         extension_count++;
        //     }
        //     if (arg6 != NULL && extension_count < 6)
        //     {
        //         strncpy(extension_list[extension_count], arg6, 10);
        //         extension_count++;
        //     }

        //     char output_file[MAX_FILENAME];
        //     snprintf(output_file, MAX_FILENAME, "temp.tar.gz");

        //     // Create a tar file containing the files with extensions in the extension_list
        //     if (extension_count > 0 && extension_count <= 6)
        //     {
        //         tar_directory_by_extension(getenv("HOME"), output_file, extension_list, extension_count);
        //         printf("Output file created: %s\n", output_file);
        //         char *filename = "temp.tar.gz";
        //         check_tar_file(client_socket, filename);
        //         send_file(client_socket, output_file);

        //         // Delete the file using the remove() function
        //         char ackbuffer[1024] = {0};
        //         int ackread = read(client_socket, ackbuffer, 1024);

        //         printf("%s\n", ackbuffer);
        //         if (strncmp(ackbuffer, "Success3", 8) == 0)
        //         {
        //             char *temp = "temp.tar.gz";

        //             // Delete the file using the remove() function
        //             if (remove(temp) == 0)
        //             {
        //                 printf("File %s deleted successfully\n", temp);
        //             }
        //             else
        //             {
        //                 fprintf(stderr, "Error: unable to delete file %s\n", temp);
        //             }
        //         }
        //     }
        //     else
        //     {
        //         printf("No file found\n");
        //         write(client_socket, "No file found", 14);
        //     }
        // }

        ////////=============gettargz END=============================================================//////////////////
        ////////=============getfiles START=============================================================//////////////////
        else if (strncmp(buffer, "getfiles", 8) == 0)
        {
            // command filename extraction
            char *filenames[6];
            int num_filenames = 0;
            char *token = strtok(buffer + 9, " ");
            while (token != NULL && num_filenames < 6)
            {
                filenames[num_filenames++] = token;
                token = strtok(NULL, " ");
            }

            if (num_filenames == 0)
            {
                char *result = "No files specified";
                send(client_socket, result, strlen(result), 0);
            }
            else
            {
                char output_file[MAX_FILENAME];
                snprintf(output_file, MAX_FILENAME, "temp.tar.gz");

                int ret = tar_directory_with_files(getenv("HOME"), output_file, filenames, num_filenames);

                if (ret == 0)
                {
                    printf("Output file created: %s\n", output_file);
                    char *filename = "temp.tar.gz";
                    check_tar_file(client_socket, filename);
                    send_file(client_socket, output_file);

                    // remove() function to delete file
                    char ackbuffer[1024] = {0};
                    int ackread = read(client_socket, ackbuffer, 1024);

                    printf("%s\n", ackbuffer);
                    if (strncmp(ackbuffer, "Success", 8) == 0)
                    {
                        char *temp = "temp.tar.gz";

                        // remove() function to delete file
                        if (remove(temp) == 0)
                        {
                            printf("File %s deleted successfully\n", temp);
                        }
                        else
                        {
                            fprintf(stderr, "Error: unable to delete file %s\n", temp);
                        }
                    }
                }
                else
                {
                    char *result = "No file found";
                    send(client_socket, result, strlen(result), 0);
                }
            }
        }
        else if(strncmp(buffer, "quit", 4) == 0) {
            printf("Closing connection\n");
            
            break;
        }
        ////////=============getfiles END=============================================================//////////////////
        else
        {
            // command unknown
            char *result = "Unknown command";
            send(client_socket, result, strlen(result), 0);
        }
    }
}

int main()
{
    int server_socket, client_socket, address_len;
    struct sockaddr_in server_address, client_address;
    // server socket create
    if ((server_socket = socket(AF_INET, SOCK_STREAM, 0)) == -1)
    {
        perror("socket");
        exit(1);
    }
    // server socket to address and port connection
    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = INADDR_ANY;
    server_address.sin_port = htons(PORT);
    memset(&(server_address.sin_zero), '\0', 8);
    if (bind(server_socket, (struct sockaddr *)&server_address, sizeof(server_address)) == -1)
    {
        perror("bind");
        exit(1);
    }
    // await incoming connections 
    if (listen(server_socket, MAX_PENDING) == -1)
    {
        perror("listen");
        exit(1);
    }
    // client connections wait condition
    while (1)
    {
        address_len = sizeof(struct sockaddr_in);
        if ((client_socket = accept(server_socket, (struct sockaddr *)&client_address, &address_len)) == -1)
        {
            perror("accept");
            continue;
        }
        // fork child process to service client request
        if (fork() == 0)
        {
            // child doesn't need to listen for new connections
            close(server_socket); 
            processclient(client_socket);
        }
        else
        {
            // parent doesn't need client socket
            close(client_socket); 
        }
    }
    return 0;
}

void send_file(int client_socket, char *filename)
{
    int file = open(filename, O_RDONLY);
    if (file == -1)
    {
        printf("Error: cannot open file %s\n", filename);
        return;
    }

    struct stat file_stat;
    if (fstat(file, &file_stat) < 0)
    {
        printf("Error: cannot get file size.\n");
        close(file);
        return;
    }

    off_t offset = 0;
    int sent_bytes = sendfile(client_socket, file, &offset, file_stat.st_size);
    printf("bytes sent: %d", sent_bytes);
    if (sent_bytes < 0)
    {
        printf("Error: failed to send file %s.\n", filename);
        close(file);
        return;
    }

    close(file);

    printf("File %s sent successfully.\n", filename);
}
void add_file_to_tar(const char *filename, const char *tar_file)
{
    pid_t pid = fork();
    if (pid < 0)
    {
        printf("Error forking process to add file to tar archive\n");
        return;
    }
    else if (pid == 0)
    {
        execlp("tar", "tar", "-rf", tar_file, filename, NULL);
        exit(1);
    }
    else
    {
        waitpid(pid, NULL, 0);
    }
}
void tar_directory(const char *directory, const char *output_file, int size1, int size2)
{
    DIR *dir;
    struct dirent *entry;
    char path[MAX_PATH];

    dir = opendir(directory);
    if (dir == NULL)
    {
        printf("Error opening directory %s\n", directory);
        exit(1);
    }

    while ((entry = readdir(dir)) != NULL)
    {
        if (strcmp(entry->d_name, ".") == 0 || strcmp(entry->d_name, "..") == 0)
        {
            continue;
        }

        snprintf(path, MAX_PATH, "%s/%s", directory, entry->d_name);

        struct stat st;
        if (lstat(path, &st) < 0)
        {
            printf("Error getting information for file %s\n", path);
            continue;
        }

        if (S_ISDIR(st.st_mode))
        {
            tar_directory(path, output_file, size1, size2);
        }
        else if (S_ISREG(st.st_mode))
        {
            if (st.st_size >= size1 && st.st_size <= size2)
            {
                add_file_to_tar(path, output_file);
            }
        }
    }

    closedir(dir);
}
void tar_directory_by_date(char *dir_path, char *output_file, char *date1, char *date2)
{
    // store the list of files by creating a temporary file
    char temp_file[MAX_FILENAME];
    snprintf(temp_file, MAX_FILENAME, "/tmp/files_%d", getpid());

    // Search for files created between date1 and date2, and write their paths to a temporary file
    char command[MAX_BUFFER];
    snprintf(command, MAX_BUFFER, "find %s -type f -newermt \"%s\" ! -newermt \"%s\" > %s", dir_path, date1, date2, temp_file);
    system(command);

    // Create a tar file containing the files listed in the temporary file
    snprintf(command, MAX_BUFFER, "tar -czf %s -T %s", output_file, temp_file);
    system(command);

    // Delete the temporary file
    snprintf(command, MAX_BUFFER, "rm %s", temp_file);
    system(command);
}
