
# Connect Service DB to RDS of AWS

For connect one service to RDS of the aws you need create the next service configuration:

    apiVersion: v1
    kind: Service
    metadata:
      name: postgresql-rds
      namespace: "example-staging"
      labels:
        app.kubernetes.io/name: postgresql
        app: example-backend
        run: example
    spec:
      externalName: XXXXXXXXXXX.eu-south-1.rds.amazonaws.com
      selector:
        app: postgresql
      type: ExternalName
    status:
      loadBalancer: {}

