
# Connect with the cluster

?> **Tip** You need to have the following keys at your disposal. You can ask the project administrator.

    export AWS_ACCESS_KEY_ID=""
    export AWS_SECRET_ACCESS_KEY=""
    export AWS_DEFAULT_REGION="eu-south-1"    

Login with AWS services

    aws configure --profile example

Check available clsuters:

    aws eks list-clusters

Response expected

    {
        "clusters": []
    }

