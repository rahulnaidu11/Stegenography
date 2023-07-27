#include <stdio.h>
#include <string.h>
#include "encode.h"
#include "types.h"
#include "common.h"
#include <unistd.h>

/* Function Definitions */

/* Get image size
 * Input: Image file ptr
 * Output: width * height * bytes per pixel (3 in our case)
 * Description: In BMP Image, width is stored in offset 18,
 * and height after that. size is 4 bytes
 */
uint get_image_size_for_bmp(FILE *fptr_image)
{
    uint width, height;
    // Seek to 18th byte
    fseek(fptr_image, 18, SEEK_SET);

    // Read the width (an int)
    fread(&width, sizeof(int), 1, fptr_image);
    printf("width = %u\n", width);

    // Read the height (an int)
    fread(&height, sizeof(int), 1, fptr_image);
    printf("height = %u\n", height);

    // Return image capacity
    return width * height * 3;
}

/* 
 * Get File pointers for i/p and o/p files
 * Inputs: Src Image file, Secret file and
 * Stego Image file
 * Output: FILE pointer for above files
 * Return Value: e_success or e_failure, on file errors
 */
Status open_files(EncodeInfo *encInfo)
{
    // Src Image file
    encInfo->fptr_src_image = fopen(encInfo->src_image_fname, "r");
    // Do Error handling
    if (encInfo->fptr_src_image == NULL)
    {
	perror("fopen");
	fprintf(stderr, "ERROR: Unable to open file %s\n", encInfo->src_image_fname);

	return e_failure;
    }

    // Secret file
    encInfo->fptr_secret = fopen(encInfo->secret_fname, "r");
    // Do Error handling
    if (encInfo->fptr_secret == NULL)
    {
	perror("fopen");
	fprintf(stderr, "ERROR: Unable to open file %s\n", encInfo->secret_fname);

	return e_failure;
    }

    // Stego Image file
    encInfo->fptr_stego_image = fopen(encInfo->stego_image_fname, "w");
    // Do Error handling
    if (encInfo->fptr_stego_image == NULL)
    {
	perror("fopen");
	fprintf(stderr, "ERROR: Unable to open file %s\n", encInfo->stego_image_fname);

	return e_failure;
  }

    // No failure return e_success
    return e_success;
}
Status read_and_validate_encode_args(char *argv[], EncodeInfo *encInfo)
{
    char *extn;
    if( strcmp(strstr(argv[2],"."),".bmp")==0)
    {
	encInfo->src_image_fname = argv[2];

    }
    else
    {
	return e_failure;
    }

    if(strcmp(extn=strstr(argv[3],"."),".txt") == 0 || (strcmp(extn=strstr(argv[3],"."),".c") == 0))
    {
	encInfo->secret_fname=argv[3];
	strcpy(encInfo->extn_secret_file,extn);
    }
    else
    {
	return e_failure;
    }
    if(argv[4]!=NULL)
    {
	encInfo->stego_image_fname=argv[4];
    }
    else
    {
	encInfo->stego_image_fname="stego_output.bmp";
    }
    return e_success;


}
Status check_capacity(EncodeInfo *encinfo)
{
    encinfo->image_capacity = get_image_size_for_bmp(encinfo->fptr_src_image);
    encinfo->size_secret_file= get_file_size(encinfo->fptr_secret);
//    printf("File size is =%ld",encinfo->size_secret_file);
    if((encinfo->image_capacity) > (54 +( strlen("#*") + 4 + 4 + 4 + encinfo->size_secret_file)*8))
	return e_success;
    else
	return e_failure;
}
Status do_encoding(EncodeInfo *encinfo)
{
    if(open_files(encinfo) == e_success)
    {
	if(check_capacity(encinfo) == e_success)
	{
	    sleep(1);
	    printf("INFO: Check capacity completed successfully\n");

	    printf("\n");
	    if( copy_bmp_header(encinfo->fptr_src_image,encinfo->fptr_stego_image) == e_success)
	    {
		sleep(1);
		printf("INFO: Header copy succsefull\n");
		printf("\n");
		if(encode_magic_string(MAGIC_STRING,encinfo)==e_success)
		{
		    sleep(1);
		    printf("INFO: Magic String copied successfully\n");
		    printf("\n");
		    if(secret_file_extn_size(strlen(encinfo->extn_secret_file),encinfo)==e_success)
		    {
			sleep(1);
			printf("INFO: File Extension size encoded Sucessfully\n");
			printf("\n");
			if(encode_secret_file_extn(encinfo->extn_secret_file,encinfo)==e_success)
			{
			    sleep(1);
			    printf("INFO: %s Encoded successfully\n",encinfo->extn_secret_file);
			    printf("\n");
			    if(encode_secret_file_size(encinfo->size_secret_file,encinfo)==e_success)
			    {
				sleep(1);
				printf("INFO: File data size encoded successfully\n");
				printf("\n");
				if(encode_secret_file_data(encinfo)==e_success)
				{
				    sleep(1);
				    printf("INFO: Secret Data encoded succesfully\n");
				    printf("\n");
				   if(copy_remaining_img_data(encinfo->fptr_src_image,encinfo->fptr_stego_image)==e_success)
				   {
				       sleep(1);
				       printf("INFO: Enocoding Completed Successfully\n");
				       printf("\n");
				   }
				   else
				       printf("ERROR: Remaining data failed to encode");

				}
				else
				    printf("ERROR: Secret data failed to encode");
			    }
			    else
				printf("ERROR: Secret file data size failed to encode");

			}
			else
			    printf("ERROR: Secret file extension failed to encode");
		    }
		    else
			printf("ERROR: Secret file extension size failed to encode");

		}

		else
		{
		    return e_failure;
		}
	    }
	}
    }
}
uint get_file_size(FILE *fptr_secret)
{
    fseek(fptr_secret,0,SEEK_END);
    return ftell(fptr_secret);

}
Status copy_bmp_header(FILE *fptr_src_image, FILE *fptr_dest_image)
{
    char str[54];
    rewind(fptr_src_image);
    fread(str,sizeof(str),1,fptr_src_image);
    fwrite(str,54,1,fptr_dest_image);
    return e_success;
}
Status encode_magic_string(char *magic_string, EncodeInfo *encinfo)
{
    if(encode_data_to_image(magic_string,strlen(magic_string),encinfo->fptr_src_image,encinfo->fptr_stego_image) == e_success)
    {
	return e_success;
    }

}
Status encode_data_to_image(char *data, int size, FILE *fptr_src_image, FILE *fptr_stego_image)
{
    char image_buffer[8];

    for( int i=0 ; i < size ; i++)
    {
	fread(image_buffer,sizeof(image_buffer),1,fptr_src_image);
	encode_byte_to_lsb(data[i],image_buffer);
	fwrite(image_buffer,8,1,fptr_stego_image);
    }
    return e_success;
}

Status encode_byte_to_lsb(char data, char *image_buffer)
{
    for(int i=0; i<8 ; i++)
    {
	image_buffer[i] = (((data & ( 1 << (7-i))) >> (7 - i)) | (image_buffer[i] & (~1)));
    }
}

Status secret_file_extn_size(int size,EncodeInfo *encinfo)
{
    char image_buffer[32];
 //   printf("%d",size);
 //   printf("VALUE OF :::%d\n", ftell(encinfo -> fptr_src_image));
    fread(image_buffer,sizeof(image_buffer),1,encinfo->fptr_src_image);
  //  printf("VALUE OF :::%d\n", ftell(encinfo -> fptr_src_image));
    encode_size_lsb(size,image_buffer);
   // printf("VALUE OF :::%d\n", ftell(encinfo -> fptr_src_image));
    fwrite(image_buffer,32,1,encinfo->fptr_stego_image);
    return e_success;
}
Status encode_size_lsb(int size, char *image_buffer)
{
    for(int i=0 ; i < 32 ; i++)
    {
	image_buffer[i] = (((size & ( 1 << (31 -i))) >> (31 - i)) | (image_buffer[i] & (~1)));

    }
    
    return e_success;
}
Status encode_secret_file_extn(char *file_extn, EncodeInfo *encinfo)
{
 //   printf("VALUE OF before extn%d\n", ftell(encinfo -> fptr_src_image));
    if(encode_data_to_image(file_extn,strlen(file_extn),encinfo->fptr_src_image,encinfo->fptr_stego_image)==e_success)

   // printf("VALUE OF after file extn%d\n", ftell(encinfo -> fptr_src_image));
	return e_success;
}
Status encode_secret_file_size(long file_size, EncodeInfo *encinfo)
{
    secret_file_extn_size(file_size , encinfo);
    return e_success;
}
Status encode_secret_file_data(EncodeInfo *encInfo)
{
    char str[encInfo->size_secret_file];
    rewind(encInfo->fptr_secret);
    fread(str,encInfo->size_secret_file-1,1,encInfo->fptr_secret);
    encode_data_to_image(str,sizeof(str),encInfo->fptr_src_image,encInfo->fptr_stego_image);
    return e_success;
}

Status copy_remaining_img_data(FILE *fptr_src,  FILE *fptr_dest)
{
    int pos=ftell(fptr_src);
    fseek(fptr_src,pos,SEEK_END);
    int size=ftell(fptr_src)-pos;
    char str[size];
    fseek(fptr_src,-size,SEEK_CUR);
    fread(str,sizeof(str),1,fptr_src);
    fwrite(str,sizeof(str),1,fptr_dest);
    return e_success;

}



