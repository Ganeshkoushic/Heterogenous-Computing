
#####	PORTING GPU CODES TO FPGA PLATFORM IN OPENCL	#####
--> The kernel code (.cl) file remains the same 

--> There are three changes in host code (.c) files:
	1) Change parameters in clGetDeviceIDs function from gpu ? CL_DEVICE_TYPE_GPU : CL_DEVICE_TYPE_CPU to CL_DEVICE_TYPE_ALL
	2) Change parameters in LoadOpenCLKernel function from "kernel_name.cl" to "./xclbin/vector_addition.hw_emu.xilinx_aws-vu9p-f1-04261818_dynamic_5_0.xclbin"
	3) Change the function clCreateProgramWithSource() to clCreateProgramWithBinary

#####	SET UP AWS EC2 INSTANCE	    #####
--> After enabling the AWS account, select EC2 instance - AWS marketplace FPGA Developer AMI - C4.x and F1.2x instance, get the key pairs

--> To add files on any of the two instances - $ ssh -i Keyname.pem centos@PublicDNSName

#####	SDAccel GIT CLONE ON C4 AND F1 INSTANCE	   #####
--> $ ssh -i Keyname.pem centos@PublicDNSName
--> $ cd src/project_data/ 
--> $ git clone https://github.com/aws/aws-fpga.git 
--> $ cd aws-fpga
--> $ source sdaccel_setup.sh

#####	SOFTWARE AND HARDWARE EMULATION ON C4 INSTANCE (CPU)	#####
--> $ cd ..
--> $ git clone https://github.com/Xilinx/SDAccel_Examples.git 
--> $ cd SDAccel_Examples/getting_started/hello_world/
--> $ mkdir Problem1 (this directory should contain .c file, .cl file, image 
--> $ cd hello_world/hello_world_ocl 
--> Copy "utils.mk", "Makefile", "sdaccel_profile_summary.csv" and "sdaccel_profile_summary.html" files from hello_world_ocl directory to your directory (Problem1) - you can do this using Filezilla

### CHANGES IN MAKEFILE	###
--> Change C file name:  HOST_SRCS += src/host.cpp to HOST_SRCS += GEMM.c (for Direct convolution, the file name will be different like DirectConvolution.c)
--> Change kernel .cl file name: $(XCLBIN)/vector_add.$(TARGET).$(DSA).xo: vector_addition.cl to $(XCLBIN)/vector_add.$(TARGET).$(DSA).xo: GEMM.cl
--> Change the kernel name: $(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_vector_addition) -c -k vector_add -I'$(<D)' -o'$@' '$<'
				$(XCLBIN)/vector_addition.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_vector_addition_OBJS)
				mkdir -p $(XCLBIN)
				$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_vector_addition) -l $(LDCLFLAGS) --nk vector_add:1 -o'$@' $(+) 
				--to-- 
			    $(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_vector_addition) -c -k convolute -I'$(<D)' -o'$@' '$<'
			    	$(XCLBIN)/vector_addition.$(TARGET).$(DSA).xclbin: $(BINARY_CONTAINER_vector_addition_OBJS)
		            	mkdir -p $(XCLBIN)
			    	$(XOCC) $(CLFLAGS) --temp_dir $(BUILD_DIR_vector_addition) -l $(LDCLFLAGS) --nk convolute:1 -o'$@' $(+)
--> Rest of the Makefile remains the same

### SOFTWARE EMULATION ###
--> $ cd Problem1/ (This folder now has .c file, .cl file, image, "utils.mk", "Makefile", "sdaccel_profile_summary.csv" and "sdaccel_profile_summary.html")
--> Change the host code "xclbin" name from "./xclbin/vector_addition.hw_emu.xilinx_aws-vu9p f1-04261818_dynamic_5_0.xclbin" to "./xclbin/vector_addition.sw_emu.xilinx_aws-vu9p-f1-04261818_dynamic_5_0.xclbin"
--> $ make check TARGET=sw_emu PROFILE=system REPORT=estimate DEVICE=$AWS_PLATFORM all 

### HARDWARE EMULATION ###
--> After successfully doing software emulation, again change the host code "xclbin" name from "./xclbin/vector_addition.sw_emu.xilinx_aws-vu9p f1-04261818_dynamic_5_0.xclbin" to "./xclbin/vector_addition.hw_emu.xilinx_aws-vu9p-f1-04261818_dynamic_5_0.xclbin"
--> $ make check TARGET=hw_emu PROFILE=system REPORT=estimate DEVICE=$AWS_PLATFORM all 
--> This will take from 10-20 hours depending on the complexity of your OpenCL code

#####	 HARDWARE SYNTHESIS ON C4 INSTANCE	#####
--> After software and hardware emulation, this is the last step on C4 instance. 
--> First, change the host code "xclbin" name from "./xclbin/vector_addition.hw_emu.xilinx_aws-vu9p f1-04261818_dynamic_5_0.xclbin" to "./xclbin/vector_addition.hw.xilinx_aws-vu9p-f1-04261818_dynamic_5_0.xclbin"
--> $ make TARGETS=hw DEVICES=$AWS_PLATFORM all
--> This will take from 3-8 hours depending on the complexity of your OpenCL code

### CREATING AWD FPGA BINARY and AFI FROM XCLBIN ###
--> After software and hardware emulation and hardware synthesis, xclbin directory will be created in Problem1 directory. 
--> $ cd xclbin/
--> $ aws configure
--> This command will ask you for 4 things, just add them as given below:
	$ AWS Access Key ID: AKIAI7QKJ5JZWSBRT52A
	$ AWS Secret Access Key: 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
	$ Default Region Name: us-east-1
	$ Default output format: json
--> After this, run the following command $ /home/centos/src/project_data/aws-fpga/SDAccel/tools/create_sdaccel_afi.sh -xclbin=/home/centos/src/project_data/SDAccel_Examples/getting_started/hello_world/Problem1/xclbin/vector_addition.hw.xilinx_aws-vu9p-f1-04261818_dynamic_5_0.xclbin -s3_bucket=aws-hw1 -s3_dcp_key=dcp -s3_logs_key=logs
--> This will create the awsxclbin file in xclbin directory of Problem1 directory

#####	FINAL RUN ON F1 INSTANCE	#####
--> $ ssh -i Keyname.pem centos@PublicDNSName - do this on F1 instance
--> $ cd src/project_data/ 
--> $ git clone https://github.com/aws/aws-fpga.git 
--> $ cd aws-fpga
--> $ source sdaccel_setup.sh
--> $ cd .. 
--> $ mkdir Problem1_F1
--> Copy the executable file in Problem1 directory of C4 instance, host .c file and .cl kernel file, image and xclbin directory of Problem1 directory on the F1 instance using Filezilla on Problem1_F1
--> $ cd Problem1_F1/xclbin/
--> $ sudo sh
--> $ source /opt/xilinx/xrt/setup.sh 
--> $ ./executable

