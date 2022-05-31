# Create CI Pipeline using GITHUB ACTIONS

## Automate your workflow from idea to production

[GitHub Actions](https://github.com/features/actions?utm_source=google&utm_medium=ppc&utm_campaign=2022q3-adv-WW-Google_Search-eg_brand&scid=7013o000002CdxYAAS&gclid=Cj0KCQjw-daUBhCIARIsALbkjSb3hXXq48se14kox3karyI4BEqQd8IonBpv1jt9PpfAekhtce9fOvoaAh7jEALw_wcB) makes it easy to automate all your software workflows, now with world-class CI/CD. Build, test, and deploy your code right from GitHub. Make code reviews, branch management, and issue triaging work the way you want.


![k8s](https://geertbaeke.files.wordpress.com/2021/01/actions.png ':size=100%')

Using this steps is possible create one pipeline for the BUILD/DEPLOY/UPDATE_DB steps

## Create the pipeline

Create the next file inside of the directory when you have the service for create the build image

    .github/workflows/deploy_production.yml

![pipeline](docs/k8s_canary_postgres_redis_rds/pipelineImage.png ':size=100%')

Add the next content inside of the file and replace the variables and names

    name: "Deploy Production EKS-K8s (CI)"


    on:
      push:
        branches: [ production ]
      pull_request:
        branches: [ production ]

    # THIS CONFIGURATION USE THE  ACCESS KEY ID:
    env:
      RELEASE_REVISION: "pr-${{ github.event.pull_request.number }}-${{ github.event.pull_request.head.sha }}"
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
      KUBE_NAMESPACE: example-${{ github.ref_name }}
      KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
      ECR_REPOSITORY: backend
      SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      VERSION_TAG: "${{ github.ref_name }}_1.${{ github.run_number }}"
      VERSION_TAG_LATEST: "${{ github.ref_name }}_latest"
    
    jobs:
      Build-Image:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v1
        - name: Use Node.js ${{ matrix.node-version }}
          uses: actions/setup-node@v1
          with:
            node-version: ${{ matrix.node-version }}
        - name: LS FLIES
          run:  ls
        - name: Create the .env File 
          run: echo "${{secrets.ENV_PRODUCTION }}" | base64 --decode > .env.production
          
        - name: Check if exist file .env
          run: ls -a
    
        - name: Build docker
          run:  docker build -t example/backend:latest .
    
        - name: docker tag
          run: docker tag example/backend:latest ghcr.io/example/backend:"${{ env.VERSION_TAG_LATEST }}"
        - name: docker tag
          run: docker tag example/backend:latest ghcr.io/example/backend:"${{ env.VERSION_TAG }}"
    
        - name: Log in to the Container registry
          uses: docker/login-action@v1
          with:
            registry: ghcr.io
            username: ${{ github.actor }}
            password: ${{ secrets.GITHUB_TOKEN }}
    
        - name: Pull Latest Image from ghcr.io
          run: docker push ghcr.io/example/backend:"${{env.VERSION_TAG_LATEST}}"
    
        - name: Pull version Image from ghcr.io
          run: docker push ghcr.io/example/backend:"${{env.VERSION_TAG}}"
    
    
      deploy:                                       
        name: Deploy                                
        runs-on: ubuntu-latest    
        needs: [Build-Image]                   
        steps:                                       
          - name: Cancel Previous Runs               
            uses: styfle/cancel-workflow-action@0.4.1
            with:                                    
              access_token: ${{ github.token }}
    
          - name: Checkout                                  
            uses: actions/checkout@v2                       
            with:                                           
              ref: ${{ github.event.pull_request.head.sha }}
    
          - name: Configure AWS credentials                          
            uses: aws-actions/configure-aws-credentials@v1           
            with:                                                    
              aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}        
              aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
              aws-region: ${{ env.AWS_REGION }}
    
          - name: Output Run ID
            run: echo ${{ github.run_id }}
          - name: Output Run Number
            run: echo ${{ github.run_number }}
          - name: Output Run Attempt
            run: echo ${{ github.run_attempt }}
    
          - name: Login to Amazon ECR            
            id: login-ecr                        
            uses: aws-actions/amazon-ecr-login@v1
    
          - name: Set Image to Kubernetes cluster                                                                            
            uses: kodermax/kubectl-aws-eks@master                                                                         
            env:                                                                                                          
              RELEASE_IMAGE: ghcr.io/example/backend:"${{env.VERSION_TAG}}"
              KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
              KUBECTL_VERSION: "v1.23.0"
            with:                                                                                                         
              args: set image deployment/example-backend example-backend=${{ env.RELEASE_IMAGE }} --record -n $KUBE_NAMESPACE   
          
          - name: Verify Kubernetes deployment                               
            uses: kodermax/kubectl-aws-eks@master     
            env:                                                                                                          
              RELEASE_IMAGE: ghcr.io/example/backend:"${{ github.ref_name }}"
              KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}              
              KUBECTL_VERSION: "v1.23.0"         
            with:                                                            
              args: -n example-production set image deployment/example-backend example-backend=ghcr.io/example/backend:"${{env.VERSION_TAG}}"
    
                    
    
      update-db:                                       
        name: UpdateDB                                
        runs-on: ubuntu-latest    
        needs: [Deploy]                   
        steps:                                       
          - name: Cancel Previous Runs               
            uses: styfle/cancel-workflow-action@0.4.1
            with:                                    
              access_token: ${{ github.token }}
    
          - name: Checkout                                  
            uses: actions/checkout@v2                       
            with:                                           
              ref: ${{ github.event.pull_request.head.sha }}
    
          - name: Configure AWS credentials                          
            uses: aws-actions/configure-aws-credentials@v1           
            with:                                                    
              aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}        
              aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
              aws-region: ${{ env.AWS_REGION }}
    
          - name: Login to Amazon ECR            
            id: login-ecr                        
            uses: aws-actions/amazon-ecr-login@v1
    
          - name: Install Dev dependecies                                                                         
            uses: kodermax/kubectl-aws-eks@master                                                                         
            env:                                                                                                          
              RELEASE_IMAGE: ghcr.io/example/backend:"${{ github.ref_name }}"
              KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
              KUBECTL_VERSION: "v1.23.0"
              args: exec -it svc/example-backend -n $KUBE_NAMESPACE  -- npm install --dev
    
          - name: Migrate DB                                                                          
            uses: kodermax/kubectl-aws-eks@master                                                                         
            env:                                                                                                          
              RELEASE_IMAGE: ghcr.io/example/backend:"${{ github.ref_name }}"
              KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
              KUBECTL_VERSION: "v1.23.0"
              args: exec -it svc/example-backend -n $KUBE_NAMESPACE  -- npx sequelize-cli db:migrate sequelize init
    
    
          - name: RUN SEEDERS DB                                                                          
            uses: kodermax/kubectl-aws-eks@master                                                                         
            env:                                                                                                          
              RELEASE_IMAGE: ghcr.io/example/backend:"${{ github.ref_name }}"
              KUBE_CONFIG_DATA: ${{ secrets.KUBE_CONFIG_DATA }}
              KUBECTL_VERSION: "v1.23.0"
              args: exec -it svc/example-backend -n $KUBE_NAMESPACE  -- npx sequelize-cli db:seed:all