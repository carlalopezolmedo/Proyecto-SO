#include <sys/types.h> 
#include <sys/socket.h> 
#include <netinet/in.h> 
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <mysql.h>
#include <pthread.h>
#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdbool.h>
#include <netinet/in.h>
#include <pthread.h>
#include <time.h>


MYSQL *conn;
int i;
int puerto = 9050;
int Socket[100];
char peticion[512];

void conectarBD() { //Conectar con la base de datos
	
	conn = mysql_init(NULL);
	if (conn==NULL) {
		printf ("Error al crear la conexion: %u %s\n",
				mysql_errno(conn), mysql_error(conn));
		exit (1);
	}
	conn = mysql_real_connect (conn, "shiva2.upc.es","root", "mysql", "JuegoPropio", 0, NULL, 0);
	//conn = mysql_real_connect (conn, "localhost","root", "mysql", NULL, 0, NULL, 0);
	if (conn==NULL)
	{
		printf ("Error al inicializar la conexion: %u %s\n",
				mysql_errno(conn), mysql_error(conn));
		exit (1);
	}

void CerrarBD() //Cerrar conexión con la base de datos
	{
	if(conn != NULL)
	{
		mysql_close(conn);
		printf ("Conexión cerrada con la base de datos.\n");
	};

int main(int argc, char *argv[])
{
	
	int sock_conn, sock_listen, ret;
	struct sockaddr_in serv_adr;
	char peticion[512];
	char respuesta[512];
	
	if((sock_listen = socket(AF_INET, SOCK_STREAM, 0)) < 0)
		printf ("Error creant el socket");
	
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family = AF_INET;
	
	serv_adr.sin_addr.s_addr = htonl (INADDR_ANY);
	
	serv_adr.sin_port = htons(9050);
	if(bind(sock_listen, (struct sockaddr *) &serv_adr, sizeof(serv_adr)) < 0)
		printf ("Error al bind");
	
	if(listen(sock_listen, 3) < 0)
		printf("Error en el listen");
	
	int i;
	
	for (i=0;i<5;i++)
	{
		printf("Escuchando\n");
		
		sock_conn = accept(sock_listen, NULL, NULL);
		printf ("He recibido conexion\n");
		
		ret=read(sock_conn,peticion,sizeof(peticion));
		printf("Recibido\n");
		
		peticion[ret]= '\0';
		
		printf("Peticion: %s\n", peticion);
		
		//Vamos a ver que quiere el cliente
		
		char *p = strtok(peticion, "/");
		int codigo = atoi (p);
		p = strtok(NULL, "/");
		char nombre[20];
		strcpy(nombre, p);
		printf("Codigo: %d, Nombre: %s\n", codigo, nombre);
		
		if (codigo == 1) //Registrar
		{
			char query[512];
			char password[20];
			p = strtok(NULL, "/");
			strcpy ( password, p);
			sprintf(query, "INSERT INTO Jugador(nombre, password) VALUES (%s, %s)", nombre, password);
			// Ejecutamos la consulta
			if (mysql_query(conn, query)) {
				printf("Error al insertar datos en la tabla: %s\n", mysql_error(conn));
				sprintf(respuesta, "Error en el registro");
			} else {
				printf("Usuario registrado con exito\n");
				sprintf(respuesta, "Registro exitoso");
			}
			
			// Enviamos la respuesta al cliente
			write(sock_conn, respuesta, strlen(respuesta));
			
		}
		
		else if (codigo == 2) //Iniciar Sesión
		{
			
			char query[512];
			char password[20];
			p = strtok(NULL, "/");
			strcpy(password, p);
			sprintf(query, "SELECT *FROM Jugador WHERE nombre=%s AND password=%s", nombre, password);
			if(mysql_query(conn,query))
			{
				printf("Error en la consulta: %s\n", mysql_error(conn));
				sprintf(respuesta, "Error en el inicio de sesion");
			}
			else
			{
				MYSQL_RES*res = mysql_store_result(conn);
				if(mysql_num_rows(res)>0)
				{
					printf("Inicio de sesion exitoso\n");
					sprintf(respuesta, "Inicio de sesion incorrectos");
				} else {
					printf("Usuario o contraseña incorrectos\n");
					sprintf(respuesta, "Usuario o contraseña incorrectos");
				}
				mysql_free_result(res);
			}
			write(sock_conn, respuesta, strlen(respuesta));
		}
		
		
		else if (codigo == 3) //Consulta
		{
			char query[512];
			sprintf(query, "SELECT * FROM Jugador WHERE nombre=%s", nombre);
			
			if(mysql_query(conn, query))
			{
				
				printf("Error en la consulta: %s\n", mysql_error(conn));
				sprintf(respuesta, "Error en la consulta");
			}else{
				MYSQL_RES*res = mysql_store_result(conn);
				MYSQL_ROW row;
				
				if((row = mysql_fetch_row(res)))
				{
					printf("Consulta exitosa\n");
					sprintf(respuesta, "Nombre: %s, Password: %s", row[1], row[2]);
				} else{
					sprintf(respuesta, "No se encontraron datos para el usuario");
				}
				mysql_free_result(res);
			}
			write(sock_conn, respuesta, srtlen(respuesta));
			
			
			
		}
		
		
		close(sock_conn);
		return 0;
	} 
}

