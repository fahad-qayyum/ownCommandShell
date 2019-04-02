
/*	Name : Fahad Qayyum
 * 	EECS login: fahadq
 * 	YU ID : 215287253
 *      Assignment # 2 Question 1
 * 	Course Code : EECS 3221
 * 	Course Name : Operating systems*/

#include <sys/stat.h>
#include <sys/types.h>
#include <ctype.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <stdbool.h>
#include <stdlib.h>
#include <unistd.h>

//---------------global variables and functions----------------

char *namedPipe = "fifo"; // pointer to pipe
int read_pipe(char *array[], bool BG);
char *command(char *array[]);
void invalid_command();
int getter(char *string, char *array[], bool *BG, bool *quit);

//----------------------Main Function--------------------------

int main(){
	
	//--------declaring variables for the function-------------
	char string[BUFSIZ] = " ";
	char *array[BUFSIZ /2 + 1];
	pid_t pid;
	int fd;	
	bool flag	= true;
	//---------------------------------------------------------
	remove(namedPipe);
	mkfifo(namedPipe, 0666);
	while(flag){
		bool BG = false;
		bool quit = false;
		printf("--SHELL--215287253>>");
		fflush(stdout);
		// reading the command and storing in string
		scanf("%[^\n]s", string);
		getchar();
		//passing the arguments to the feature
		if(getter(string, array, &BG, &quit) == 0){
			// if the command given is a quit command then teminate
			if(quit){
				printf("Terminated Successfully!\n");
				// remove the pipe to free space
				remove(namedPipe);
				exit(0);	
			}
			// if command is a background command
			if (BG){
				printf("....request submitted, returning prompt....\n");
			// if command is a foreground command	
			}else {
				printf("....working on request....");		
			}
			// creating a child process
			pid = fork();
			if(pid < 0){
				perror("Failed to fork");	
			} else if (pid == 0){	
				// in the child process
				fd = open(namedPipe, O_WRONLY);	
				// Storing in the named pipe
				dup2(fd, STDOUT_FILENO);
				dup2(fd, STDERR_FILENO);	
				// closing the pipe
				close(fd);			
				// executing the command
				if(execvp(array[0], array) == -1){
					perror("Failed to execute command!\n");	
				}	
				exit(0);				
			}else { 				
				// in the parent process read from the pipe
				read_pipe(array, BG);				
			}	
		}	
	}	
	// delete the pipe
	remove(namedPipe);
	return 0;
}

void invalid_command(){
	printf("Please give a valid command!\n");	
}

int getter(char *string, char *array[], bool *BG, bool *quit){
	
	int counter =0;
	// tokenizing the string
	char *token = strtok(string, " ");
	//  if no command typed 
	if(token == NULL){ 
		invalid_command();
		return 1;	
	}else {
		// if command is either background or foreground
		if( strcmp(token, "FG") == 0 || strcmp(token, "BG") == 0){ 
			// if its a background command
			if( strcmp(token, "BG") == 0 ){ 
				*BG = true;	
			}
			// tokenizing string
			for(counter =0 ; token!= NULL && counter < 80 ; counter++){ 
				token = strtok(NULL, " ");
				array[counter]	= token;
			}
			// prompting invalid command
			if( counter == 1){
				invalid_command();
				return 1;	
			}	
			// if command to exit the shell then make quit status true
		}	else if(strcmp(token, "exit") == 0){ 
				*quit = true;
			} else {
				// all other commands are invalid
				invalid_command(); 
				return 1;	
			}
	}
	return 0;
}

char *command(char *array[]){
	
	char * string;	
	int counter =0, j =0, k = 0;
	// allocating a pointer
	char *cmd = (char *)malloc(BUFSIZ); 
	// repeat until string is empty
	do{
		string = array[counter];
		// string manipulation
		if(string != NULL){
			for(k =0; k < strlen(string) ; k++){
				cmd[j] = string[k];	
				j++;
			}	
			cmd[j++] = ' ';
			counter++;
		}	
	} while( string != NULL && counter < BUFSIZ);
	// end of line character
	cmd[j-1] = '\0';
	return cmd;
}

int read_pipe(char *array[], bool BG){
	char choice; 
	char c;
	// opening named pipe
	int fd = open(namedPipe, O_RDONLY); 
	// if command is a background command
	if(BG){
		printf("--SHELL--215287253>>");
		char *cmd = command(array);
		printf("\n....Output for \"BG %s\" is ready. Display it now [Y/N]>>", cmd);
		free(cmd);	
	}else{
		//wait for the child process to terminate
		wait(NULL);	
		printf("\n....Output is ready. Display it now [Y/N]>>");
	}	
	// read the user choice either Y or N
	scanf("%c", &choice); 
	// if choice is not a space of a new line then read until it encounter a new line i.e. enter
	if(!isspace(choice) || choice != '\n'){ 
		while(getchar() != '\n');	
	}
	// if user chooses Y
	if(choice == 'Y'){
		while(read(fd, &c, 1) != 0){ 		
			// keep reading and printing 
			printf("%c", c);	
		}	
	}else {
			// otherwise just read and dont print
		while(read(fd, &c, 1) != 0);	
	}
		// closing file descriptor
	close(fd);
	return 0;
}
