# c-netserver-nonblocking

``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define max(a, b) ((a) > (b) ? (a) : (b))

typedef struct Client_s {
	int fd;
	char *buffer;
	size_t buffer_length;
	size_t buffer_size;
	struct sockaddr_in address;
	socklen_t address_length;
	struct Client_s *next;
} Client;

void error(char *func) {
	perror(func);
	exit(1);
}

int main(int argc, char **argv) {
	int listenfd = socket(AF_INET, SOCK_STREAM, 0);

	if(listenfd < 0)
		error("socket");

	int on = 1;

	if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
		error("setsockopt");

	struct sockaddr_in servaddr;

	memset(&servaddr, 0, sizeof(servaddr));

	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port = htons(8122);

	if(bind(listenfd, (struct sockaddr *) &servaddr, sizeof(servaddr)) < 0)
		error("bind");

	if(listen(listenfd, SOMAXCONN) < 0)
		error("listen");

	Client *client_first = NULL;

	for( ; ; ) {
		fd_set rset;

		FD_SET(listenfd, &rset);
		int maxfdp1 = listenfd;

		for(Client *client = client_first; client != NULL; client = client->next) {
			FD_SET(client->fd, &rset);
			maxfdp1 = max(maxfdp1, client->fd);
		}

		maxfdp1++;

		select(maxfdp1, &rset, NULL, NULL, NULL);

		if(FD_ISSET(listenfd, &rset)) {
			printf("accepting new socket\n");

			Client *client = calloc(1, sizeof(Client));

			if(client == NULL)
				error("calloc");

		 	client->fd = accept(listenfd, (struct sockaddr *) &client->address, &client->address_length);

			Client **client_node = &client_first;

			while(*client_node != NULL) {
				client_node = &(*client_node)->next;
			}

			*client_node = client;
		}

		for(Client *client = client_first; client != NULL; client = client->next) {
			if(FD_ISSET(client->fd, &rset)) {
				size_t buf_len = 500;
				char *buf = calloc(500, buf_len);

				recv(client->fd, buf, buf_len - 1, 0);
				printf("data from %d: %s\n", client->fd, buf);
				free(buf);
			}
		}
	}
}

```
