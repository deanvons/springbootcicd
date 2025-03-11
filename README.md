# 🚀 Step-by-Step Guide: Deploying an ASP.NET API via ACR and Automating CI/CD with GitHub Actions

This guide covers two parts:
- 1️⃣ **Manually deploying the API via Azure Container Registry (ACR) and Web App Service** (to **eliminate common issues**)
- 2️⃣ **Automating deployments with GitHub Actions** (CI/CD pipeline)

---

## **📌 Part 1: Manual Deployment via ACR and Web App Service**
This ensures everything works **before automating CI/CD**.

### **✅ Step 1: Create and Test the API**
1. **Create the API project** (ASP.NET Core Web API or Spring boot)
   ```sh
   dotnet new webapi -n MyApi
   cd MyApi
   ```
   Java: create via spring initializer and download -> add a sample endpoint (api/hello)

2. **Test the API locally**
   ```sh
   dotnet run
   ```
   Run main in Java IDE of choice
   
   Open `https://localhost:5001/weatherforecast` in a browser.
   Open `https://localhost:8080/api/hello` in a browser. For spring

3. **Create a Dockerfile** inside the API folder:
for .NET
   ```Dockerfile
   # 1 - Base image to work from. SDK for .NET
    FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-stage
    # 2 - Set working directory
    WORKDIR /app
    # 3 - Copy source files to the image
    COPY . ./
    # 4 - Run 'dotnet restore' to install missing dependencies
    RUN dotnet restore 
    # 5 - Publish our application to the image
    RUN dotnet publish -c Release -o out

    # 6 - Runtime stage
    FROM mcr.microsoft.com/dotnet/aspnet:8.0
    # 7 - re-establish the working directory
    WORKDIR /app
    # 8 - Copy over the published files (the rest will be discarded)
    COPY --from=build-stage /app/out .
    # 9 - Configure how the application is run
    ENTRYPOINT ["dotnet", "appname.dll"]
   ```

   for Java
   ```Dockerfile
   # Use a lightweight OpenJDK 21 base image
    FROM eclipse-temurin:21-jdk

    # Set the working directory inside the container
    WORKDIR /app

    # Copy the built JAR file into the container
    COPY target/[appname].jar app.jar

    # Expose the default Spring Boot port
    EXPOSE 8080

    # Run the application
    ENTRYPOINT ["java", "-jar", "app.jar"]

   ```
4. **Build and test the Docker image locally**
   ```sh
   docker build -t myapi .
   docker run -d -p 8080:80 myapi
   ```
   Open `http://localhost:8080/weatherforecast` or `http://localhost:8080/api/hello` to verify it works.

---

### **✅ Step 2: Set Up a GitHub Repository**
1. **Initialize Git**
   ```sh
   git init
   git add .
   git commit -m "Initial API setup"
   ```
2. **Create a `.gitignore` file**:
   ```
   bin/
   obj/
   .vscode/
   .idea/
   .DS_Store
   ```
3. **Push the code to GitHub**
   ```sh
   git remote add origin https://github.com/your-username/your-api-name.git
   git branch -M main
   git push -u origin main
   ```

---

### **✅ Step 3: Create and Push Image to Azure Container Registry (ACR)**
#### **1️⃣ Install and Configure Azure CLI**

Before you begin these steps ensure you have an Azure subscription. Go to Azure portal to create a free account (https://azure.microsoft.com/en-us/)
1. **Install Azure CLI**: [Download & Install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
2. **Login to Azure**
   ```sh
   az login
   ```
3. **Set your subscription** (if needed)
    You should be promted to choose a subscription

   ```sh
   az account set --subscription "your-subscription-id"
   ```

---

#### **2️⃣ Create Azure Container Registry**
```sh
az acr create --resource-group MyResourceGroup --name MyFirstRegistryAgain --sku Basic
```

or go to your portal and create it there

![Create ACR step 1](./cicdscreenshots/Create%20ACR%201.png)
![Create ACR step 2](./cicdscreenshots/Create%20ACR%202.png)
---

#### **3️⃣ Enable ACR Admin Access**
```sh
az acr update --name MyFirstRegistryAgain --admin-enabled true
```

---

#### **4️⃣ Log in to ACR**
```sh
az acr login --name MyFirstRegistryAgain
```

---

#### **5️⃣ Tag and Push the Image**
```sh
docker tag myapi:latest MyFirstRegistryAgain.azurecr.io/myapi:v1
docker push MyFirstRegistryAgain.azurecr.io/myapi:v1
```

---

#### **6️⃣ Verify Image in ACR**
```sh
az acr repository list --name MyFirstRegistryAgain --output table
```

or check it in the portal

![Create ACR step 2](./cicdscreenshots/Confirm%20Push%20to%20Container%20Registry.png)

---

### **✅ Step 4: Deploy the API to Azure Web App**
1. **Create an Azure App Service Plan**
   ```sh
   az appservice plan create --name myAppServicePlan --resource-group MyResourceGroup --sku B1 --is-linux
   ```

    or use the portal

    ![Create Service Plan](./cicdscreenshots/Create%20Service%20Plan.png)

2. **Create the Web App and Connect to ACR**
   ```sh
   az webapp create --resource-group MyResourceGroup --plan myAppServicePlan --name myapi --deployment-container-image-name MyFirstRegistryAgain.azurecr.io/myapi:v1
   ```
    


3. **Set App Service to Pull From ACR**
   ```sh
   az webapp config container set --name aspcicd --resource-group MyResourceGroup --docker-custom-image-name loopacademy.azurecr.io/aspcicd:v1
   ```
4. **Restart Web App**
   ```sh
   az webapp restart --name aspcicd --resource-group MyResourceGroup
   ```
      or use the portal
   
    ![Create Web App 1](./cicdscreenshots/Create%20Web%20app%20part%201.png)
    ![Create Web App 2](./cicdscreenshots/Create%20Web%20app%20part%202.png)


> Note: Azure can take some time so if something isn't working give it some time before you confirm and test.

5. **Test Deployment**: `[azureassignmeddomain]/weatherforecast` `[azureassignmeddomain]/api/hello`

---

## **📌 Part 2: Automating CI/CD with GitHub Actions**

### **✅ Step 5: Set Up GitHub Secrets**

To get the values for the secrets below, create an Azure Service Principal (App Registration):
1) Search for App registrations in the Azure Portal
2) Create new registration, give it a name

![Create Web App 2](./cicdscreenshots/Create%20Service%20Principle.png)

3) Leave Redirect URI blank

 ![Create Web App 2](./cicdscreenshots/Create%20Service%20Principle%202.png)

Go to **GitHub → Your Repo → Settings → Secrets and Variables → Actions** and add these secrets:

| Secret Name | Value |
|---------------|-----------|
| `AZURE_CLIENT_ID` | **Azure App Registration → Client ID** |
| `AZURE_TENANT_ID` | **Azure App Registration → Tenant ID** |
| `AZURE_SUBSCRIPTION_ID` | **Azure Subscription ID** |
| `ACR_LOGIN_SERVER` | ***Azure Portal → ACR -> Login Server** |
| `ACR_USERNAME` | **Azure Portal → ACR → Settings -> Access Keys** |
| `ACR_PASSWORD` | **Azure Portal → ACR → Settings -> Access Keys** |
| `WEBAPP_NAME` | `demo-api` |
| `RESOURCE_GROUP` | `MyResourceGroup` |

---

### **✅ Step 6: Configure GitHub Actions (CI/CD Pipeline)**
Create `.github/workflows/deploy.yml` at the root of your project:
```yaml
name: Build and Deploy to Azure

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Build and Push Docker Image
        run: |
          IMAGE_TAG=v1
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:$IMAGE_TAG -f MyProject/Dockerfile MyProject/
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:$IMAGE_TAG

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to Azure Web App
        run: |
          az webapp config container set --name ${{ secrets.WEBAPP_NAME }} --resource-group ${{ secrets.RESOURCE_GROUP }} --docker-custom-image-name ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:v1
          az webapp restart --name ${{ secrets.WEBAPP_NAME }} --resource-group ${{ secrets.RESOURCE_GROUP }}
```

---

Go to azure portal -> app registrations -> yourAppRegistrationName -> Federated credentails -> create 

 ![Create Web App 2](./cicdscreenshots/Add%20federated%20client.png)

🚀 **Now, every push automatically updates the deployment to Azure!**