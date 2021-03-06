# Oracle Functions + OCI Object Storage

This example shows how [Oracle Functions](https://docs.cloud.oracle.com/iaas/Content/Functions/Concepts/functionsoverview.htm) can interact with [Oracle Cloud Infrastructure Object Storage](https://docs.cloud.oracle.com/iaas/Content/Object/Concepts/objectstorageoverview.htm) to execute operations such as putting a new object in a storage bucket, listing the objects of a storage bucket and getting the contents of a specific object. 

It uses the Resource Principal authentication provider included in the Oracle Cloud Infrastructure SDK to enable the functions to access another OCI Object Storage service. In order for this to work, you have to configure OCI to include the function in a dynamic group and then create an IAM policy to grant the dynamic group access to Object Storage service.


Individual functions cater to the put, get and list capabilities i.e. there are three functions as a part of a single application which you will deploy to Oracle Functions. These are Java functions which use the Object Storage APIs in the [OCI Java SDK](https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/javasdk.htm) for programatic interaction.

## Pre-requisites

- Object Storage: You need to create a storage bucket - please refer to the [details in the documentation](https://docs.cloud.oracle.com/iaas/Content/Object/Tasks/managingbuckets.htm?TocPath=Services%7CObject%20Storage%7C_____2)
- Ensure you are using the latest version of the Fn CLI. To update simply run the following command - `curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh`
- Oracle Functions setup: Please ensure that you have setup and configured Oracle Functions in your tenancy as per [instructions in the documentation](https://docs.cloud.oracle.com/iaas/Content/Functions/Concepts/functionsprerequisites.htm). 

Clone this repository before proceeding further and change into the top level directory

	git clone https://github.com/abhirockzz/oracle-functions-oci-object-store
	cd oracle-functions-oci-object-store

## Create an application

Create an application with required configuration - all your functions will be a part of this application

	fn create app --annotation oracle.com/oci/subnetIds='["<OCI_SUBNET_OCIDs>"]' --config NAMESPACE=<NAMESPACE> fn-object-store-app

Summary of the configuration parameters

- `OCI_SUBNET_OCIDs` - the OCID(s) of the subnet where you want your functions to be deployed
- `NAMESPACE` - Object Storage namespace (see [this](https://docs.cloud.oracle.com/iaas/Content/Object/Tasks/understandingnamespaces.htm))

For e.g.

	fn create app --annotation oracle.com/oci/subnetIds='["ocid1.subnet.oc1.phx.aaaaaaaabrg5uf2uzc3ni4jkz4vhqwprofmlmo2mpumnuddd7iandsfoobar"]' --config NAMESPACE=foobar fn-object-store-app

## Deploy the functions

Use the command below to deploy *all* the functions (notice `--all` the the end).

	fn -v deploy --app fn-object-store-app --all

> If you want to deploy one function at a time, enter the respective directory and use the same command as above **without the `--all` directive**

To confirm successful deployment of the app and all its functions: 

	//check your app (and its config)
	fn inspect app fn-object-store-app

	//check associated functions
	fn list functions fn-object-store-app

## Oracle Functions Resource Principals configuration

### List objects function

- Get the function OCID using this command: `fn inspect fn fn-object-store-app listobjects id`
- Create a dynamic group `fn-obj-store-list-dg` with the matching rule `resource.id = '<FUNCTION_OCID>'`
- Create an IAM policy `fn-object-store-list-policy` with the below statement - please replace placeholders for `<COMPARTMENT_NAME>` and `<BUCKET_NAME>` with values specific to your OCI tenancy

		allow dynamic-group fn-obj-store-list-dg to read objects in compartment <COMPARTMENT_NAME> where all{target.bucket.name='<BUCKET_NAME>'}

### Get object function

- Get the function OCID using this command: `fn inspect fn fn-object-store-app getobject id`
- Create a dynamic group `fn-obj-store-get-dg` with the matching rule `resource.id = '<FUNCTION_OCID>'`
- Create an IAM policy `fn-object-store-get-policy` with the following statements - please replace placeholders for `<COMPARTMENT_NAME>` and `<BUCKET_NAME>` with values specific to your OCI tenancy

		allow dynamic-group fn-obj-store-get-dg to read buckets in compartment <COMPARTMENT_NAME>
	
		allow dynamic-group fn-obj-store-get-dg to read objects in compartment <COMPARTMENT_NAME> where all{target.bucket.name='<BUCKET_NAME>'}

### Put object function

- Get the function OCID using this command: `fn inspect fn fn-object-store-app putobject id`
- Create a dynamic group `fn-obj-store-put-dg` with the matching rule `resource.id = '<FUNCTION_OCID>'`
- Create an IAM policy `fn-object-store-put-policy` with the following statements - please replace placeholders for `<COMPARTMENT_NAME>` with the value specific to your OCI tenancy

		allow dynamic-group fn-obj-store-put-dg to read buckets in compartment <COMPARTMENT_NAME> 

		allow dynamic-group fn-obj-store-put-dg to manage objects in compartment <COMPARTMENT_NAME> where any {request.permission='OBJECT_CREATE', request.permission='OBJECT_INSPECT'}

## Testing

You can now test the app.

> Note: This example works with `String` data type for the `content` attribute

### Put object

To store a file with text content, use the below command

	echo -n '{"name": "<filename>", "bucketName":"<BUCKET_NAME>", "content": "<TEXT_CONTENT>"}' | fn invoke fn-object-store-app putobject

e.g.

	echo -n '{"name": "file1.txt", "bucketName":"test", "content": "This file was created in OCI object storage bucket using Oracle Functions"}' | fn invoke fn-object-store-app putobject

### Get object

To fetch the contents of a file in a bucket (the one you stored in the previous step)

	echo -n '{"name": "<filename>", "bucketName":"<BUCKET_NAME>"}' | fn invoke fn-object-store-app getobject

e.g. to get the content of the file you stored in the previous step

	echo -n '{"name": "file1.txt", "bucketName":"test-bucket"}' | fn invoke fn-object-store-app getobject

You should get the contents of the file in the response

	This file was created in OCI object storage bucket using Oracle Functions

### List object(s)
	
Finally, to list the files (file names) in a bucket

	echo -n '<BUCKET_NAME>' | fn invoke fn-object-store-app listobjects

e.g. to list the file names in bucket `test-bucket` 

	echo -n 'test-bucket' | fn invoke fn-object-store-app listobjects

you should see a response with list of all file/objects (names)

	[
	    "file1.txt",
	    "lorem.txt",
	    "README.md"
	]

