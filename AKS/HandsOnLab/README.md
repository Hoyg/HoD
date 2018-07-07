# Azure Kubernetes Service & Azure Dev Spaces

Azure Kubernetes Service (AKS) gives developers the best experience for building microservices in any platform - Java, .NET Core, or Node.js, to name a few used in this demo's source code - using Kubernetes and containers. The diagram below shows a high-level snapshot of the back-end APIs housed in the AKS cluster once you deploy this repository's source code to AKS. 

![AKS Cluster](../media/cluster.png)

[Azure Dev Spaces](https://docs.microsoft.com/en-us/azure/dev-spaces/azure-dev-spaces) provides a rapid, iterative Kubernetes development experience for teams. With minimal dev machine setup, you can iteratively run and debug containers directly in Azure Kubernetes Service (AKS). Develop on Windows, Mac, or Linux using familiar tools like Visual Studio, Visual Studio Code, or the command line. The diagram below shows how the Visual Studio family of IDEs can connect to AKS to enable debugging within developers' Azure Dev Spaces without impacting production or teammate code. 

![Dev Spaces](../media/dev-spaces.png)

The AKS Cluster created by the demo contains support for Azure Dev Spaces, so that you can debug the individual services live in the Kubernetes cluster. There's a pre-wired error in the Hotels microservice you'll fix during the demo, then debug in your own Azure Dev Space to validate the fix worked. 

# Getting Started

This lab should take 60-90 minutes to complete. 

All of the back-end systems run inside of Docker containers. During the installation phase you will notice errors if you haven't set your Docker configuration to use 4 GB of memory. Changing this is simple within the Docker configuration dialog. Just set the memory higher and restart Docker.

## Prerequisites 

* Bash command line, which can be accomplished natively on Mac or Linux or using [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) on Windows.
* [Docker](http://www.docker.com) to build the containers. 
* [Visual Studio 2017 Preview](https://www.visualstudio.com/vs/preview/) with the ASP.NET Web workload installed. 
* [Azure Dev Spaces Extension for Visual Studio](https://docs.microsoft.com/en-us/azure/dev-spaces/get-started-netcore-visualstudio#get-the-visual-studio-tools). 
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* [Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* The [jq](https://stedolan.github.io/jq/) package for bash, which enables jQuery processing. 
* [Helm](https://helm.sh/) and [Draft](https://github.com/Azure/draft) to ease Kubernetes deployment

## Set up a Service Principal 

[Create a service principal](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-service-principal-portal?view=azure-cli-latest) and take note of the Application ID and key. The service principal will need to be added to the **Contributor** for the subscription. If you already have a service principal, you can re-use it, and if you don't and create one for this demo, you can re-use it to create other AKS clusters in the future. 

## Provision the Azure Resources

In this step you'll create all of the Azure resources required by the demo. This consists of an AKS Cluster and an Azure Container Registry (ACR) instance. The AKS Cluster is pre-configured to use Microsoft Operations Management Suite (OMS) and Log Analytics to enable the rich Container Health Dashboard capabilities. 

1. Install the Azure Dev Spaces **preview** extension for the Azure CLI by entering the following command. 

    ```bash
    az extension add --name dev-spaces-preview
    ```

1. Open a bash terminal. CD into the `setup` folder of this repository. 
1. Some Linux distributions require setting execute permissions on `.sh` files prior to executing them. To be safe, running the command below results in the bash scripts being enabled with execution priveleges. 

    ```bash
    chmod +x ./00-set-vars.sh
    chmod +x ./01-aks-create.sh
    chmod +x ./02-deploy-apis.sh
    chmod +x ./03-deploy-web.sh
    chmod +x ../src/SmartHotel360-Azure-backend/deploy/k8s/build-push.sh
    chmod +x ../src/SmartHotel360-Azure-backend/deploy/k8s/deploy.sh
    ```

1. Run the command below, replacing the parameters with your own values. The script expects following parameters:

    * `-g <resource-group>`: Resource group to use
    * `-s <subscription>`: Azure Subscription to use
    * `-n <name>`: AKS cluster name to be used
    * `-r <name>`: ACR name to be used (just name, not FQDN)
    * `-l <location>`: Location to be used. Defaults to `eastus`
    * `-c <spn-client>`: Service principal app id
    * `-p <spn-pwd>`: Service principal password
    * `-a <name>`: Name of the Sh360 app to be installed in the cluster. Defaults to  `myapp`

    ```bash
    source 00-set-vars.sh -g <resource group> -s <subscription id> -n <cluster name> -r <ACR name> -l eastus -c <service principal app id> -p <service principal password>
    ```

    > **Important Note:** The only regions in which AKS and Azure Dev Spaces are currently supported are Canada East and Eaat US. So when creating a new AKS cluster for this scenario use either **canadaeast** or **eastus** for the **AKS_REGION** variable.

1. Once the script has run, create the Azure resources you'll need by running this script:

    ```bash
    source 01-aks-create.sh
    ```

Now that the AKS cluster has been created we can publish the SmartHotel360 microservice source code into it. 

## Deploy the SmartHotel360 Backend APIs

In this segment you'll build the images containing the SmartHotel360 back-end APIs and publish them into ACR, from where they'll be pulled and pushed into AKS when you do your deployment. We've scripted the complex areas of this to streamline the setup process, but you're encouraged to look in the `.sh` files to see (or improve upon) what's happening. 

> Note: Very soon, the repo will be signififcantly updated to make use of Helm as a deployment strategy. For now the scripts are mostly in bash and make use of `kubectl` to interface with the cluster. 

1. CD into the `setup` directory (if not already there) and run this command:

    ```bash
    ./02-deploy-apis.sh --httpRouting
    ```

    The script will take some time to execute, but when it is complete the `az aks browse` command will be executed and the Kubernetes dashboard will open in your browser.  Details on this script can be found [here](deploy/02-deploy-apis.md), so you can customize creation if you desire. The script above should be enough once the environment variables are set in the previous step. 

1. When the dashboard opens (you may need to hit refresh as it may 404 at first), some of the objects in the cluster may not be fully ready. Hit refresh until these are all green and at 100%. 

    ![Waiting until green](../media/still-yellow.png)

1. Within a few minutes the cluster will show 100% for all of the objects in it. 

    ![All ready](../media/all-green.png)

Congratulations! You've deployed the APIs. You're 75% of the way there, now, and all that remains is to deploy the public web site. This is a good opportunity for a much-earned break!

## Deploy the Public Web App

Now that the back-end APIs are in place the public web app can be pushed into the cluster, too. The spa app makes calls to the APIs running in the cluster and answers HTTP requests at the ingress URL you used earlier.

1. The end-to-end setup script makes use of some of the `export` environment variables you set earlier, so make sure they're still set by using the `echo` command to make sure they're still set. If you don't see values when you `echo` these environment variables, re-run `setup/00-set-vars.sh`. 

1. CD into the `setup` directory if you're not already there, and execute the command below. 

    ```bash
    ./03-deploy-web.sh --httpRouting
    ```

    The command may take a few minutes to complete. 

1. Once the command completes, execute the command below to see all of the ingresses in the AKS cluster. 

    ```bash
    kubectl get ingresses
    ```

    You'll see a list of service ingresses that have been deployed into the cluster, and each should have a public IP address assigned to it.

    ![Ingresses](../media/ingresses.png)

1. One of the ingresses has the suffix **sh360-web** - that's the public web site ingress (the prefix is dynamically generated by Kubernetes). Copy the IP address from the terminal window.

    > Note: You'll use the ingress URL later in the lab so save it in a text file somewhere, or you could use the `kubectl get ingresses` command to obtain the URL again later. 

    ![Copy ingress](../media/copy-fqdn.png)

1. Open a browser window and paste the URL into a browser and the public site should appear. 

    ![Public web](../media/public-web.png)

## Save the Queries

There are three queries provided in the [`queries`](../queries) folder of this repository:

* CPU chart of the entire cluster over time
* Error log containing "0 results found" log entry
* Bar chart over time of the error log containing "0 results found" log entry

To make it easy to run these queries during a demo, paste them in the Log Analytics Query Explorer and click the **Save** button, then give the query a name and category. 

![Save the query](../media/save-queries.png)

Then they're readily available in the **Saved Queries** folder in the Query Explorer. 

![Running the query](../media/saved-queries-running.png)

## Preload the logs with data

Later during the lab you will use the AKS logs to find and fix a bug in the site. To generate these logs, a Node.js script has been provided. It simply hits the Hotels API in the AKS cluster in a variety of ways - searching for a city, searching for hotels, and so on. The code simply repeats every 10 seconds, but you can tweak this if you like. 

### Customizing the Script for your Cluster(s)
The code for the preload is contained in [`preloader/app.js`](../preloader/app.js), and requires one minor customization. The first line of code in [`preloader/app.js`](../preloader/app.js) is an array variable of URL bases (if you're doing this as demo it never hurts to have a backup environment). You will replace this URL with the public ingress URL you copied earlier. If you've misplaced the ingress URL you can obtain it again by typing `kubectl get ingresses` at the bash command prompt. 

```javascript
const urlBases = ["http://sh360.<guid>.<region>.aksapp.io/"]
```

You'd simply change the URL to be that of your own ingress front door. An example setup might consist of the following:

```javascript
const urlBases = ["http://sh360app.f248e5d3c63241532fcb.canadaeast.aksapp.io/", "http://sh360app.f318e5d3b63240f19fcb.eastus.aksapp.io/"];
```

This would result in log and CPU data being generated for two clusters, where that in East Canada is the primary and that in East US is the secondary. 

### Running the script

Running the script is easy. Simply CD into the `preloader` directory and enter the commands below. 

```
npm install
node app
```

Allow the script to run for some time, then close it. Then repeat until you have the amount of data you want for your demo. 

> Note: Don't run the script and then expect the logs to appear immediately. Sometimes it takes 5-10 minutes for logs to appear in the portal. 

As the scipt loops, it will output the request/response data so you can see how things are going. 

![Preloader running](../media/preloader-running.png)

## The Hotels API Bug

At this point, your AKS cluster has been created and you've generated some log data from calling the Hotels API repeatedly. Now, you'll use the log data to find a bug in the Hotels API, and then you'll use the Kubernetes Tools for Visual Studio and Azure Web Spaces to fix and debug the API's code. 

1. Open the public web site in a browser using the public ingress URL you copied earlier. 
1. Search for **New** (note the capitalization) using the city search feature on the public web site. 

    ![Search working](../media/demo-script/01-city-search-good.png)

1. Search for **Sea** or **Seattle** to see that no results are found. 

    ![Nothing in Seattle](../media/demo-script/02-city-search-bad.png)

1. Flip to the AKS overview page for the demo cluster. 

    ![The AKS Cluster in the portal](../media/demo-script/03-cluster.png)

1. Click the **Health** link in the portal.

    ![Health view](../media/demo-script/04-health-view.png)

1. Click the **Containers** navigation item in the toolbar to switch to Container view. 

    ![Container view](../media/demo-script/05-container-view.png)

1. Select **default** from the namespace menu.

    ![Select namespace](../media/demo-script/06-select-namespace.png)

1. Select **hotels** from the service menu. 

    ![Select service](../media/demo-script/07-select-service.png)

1. Click on the **View Logs** link for the **Hotels** service. 

    ![Specific container view](../media/demo-script/07-view-logs.png)

1. Wait for the default query to run. 
1. If no results are shown in the results pane, increase the parameter for the `ago` method to a greater dureation, like `ago(24)` for the past 24 hours of log data. 

    ![Log data](../media/demo-script/08-health-view.png)

1. Expand one of the "*City search returned 0 results*" log entries. 

    ![Empty city searches](../media/demo-script/09-no-results.png)

1. Expand one of the `CitiesController` log entries. 

    ![Find the buggy controller](../media/demo-script/10-logs-help-the-final-view.png)

1. Expand the **Saved Queries** in the **Query Explorer** in the Azure portal, and click the **CPU over time** query. 

    ![CitiesController](../media/demo-script/10-logs-help-cpu-over-time.png)

1. Click the **Run** button. 

    ![CPU over time](../media/demo-script/10-logs-help-cpu-query-visible.png)

1. Click on the **Logs** query. 
1. Click the Run button. This query shows me the logs with the "returned 0 results" string, which we've already identified as the source of the bug. This last query might give me a more friendly view of the frequency of the failing search. 
    
    ![Log results](../media/demo-script/10-logs-help-logs-query.png)

1. Click on the **Logs Chart** query. 
1. Click the Run button.
    
    ![Log chart](../media/demo-script/10-logs-help-logs-timechart.png)

1. Flip over to Visual Studio, where the solution should be open. 
    
    ![Visual Studio](../media/demo-script/11-vs-solution-open.png)

1. Hit `Ctrl-T` to open the Visual Studio `Go to all` search pane. 
1. Type `CitiesController`.
1. Find the `CitiesController` class and click on it. 
    
    ![Searching for a type](../media/demo-script/12-vs-ctrl-t-to-search.png)

1. Right-click the `GetDefaultCities` method and select `Go to Definition`. 

    ![Go to definition](../media/demo-script/13-get-method.png)

1. Scroll up to show the `Get` method. 
    
    ![The Get method](../media/demo-script/14-both-methods.png)
    
1. Find the line of code in `CitiesController` using the `Where` method:

    ```csharp
    _citiesQuery.GetDefaultCities().Result
        .Where(city => city.Name.StartsWith(name));
    ```

1. Change the code to match this:

    ```csharp
    await _citiesQuery.Get(name);
    ```

    ![Get method](../media/demo-script/15-fixed-code.png)

1. Add a breakpoint on the `return Ok(cities);` line of code. 

    ![Add breakpoint](../media/demo-script/22-breakpoint.png)

1. Right-click the `SmartHotel.Services.Hotels` Visual Studio project. 

    ![Right-click VS Project](../media/demo-script/16-right-click-props.png)

1. Select **Properties** from the flyout menu. 
1. Select **Debug** in the left navigation. 
1. Select **Azure Dev Spaces** from the Profile menu. 

    ![Selecting Dev Spaces](../media/demo-script/17-select-dev-spaces.png)

1. Click the **Change** button. 
1. Select the AKS Cluster you want to use. 
1. Select **Create New Space** from the Space menu. 

    ![Create a new Space](../media/demo-script/18-create-new-space.png)

1. Give your space a name that doesn't already exist in the cluster.
    
    ![Name your space](../media/demo-script/19-new-space-name.png) 

1. Enable the **Launch browser** checkbox.
1. Paste in the URL of the site (this should be the web site's public URL running in your cluster).
1. Prefix the URL with the name of the space and `.s.` so that the format is `http://{yourspace}.s.{original URL}`. 

    ![Prefixing site URL with space name](../media/demo-script/20-launch-url-with-space.png)

1. Select **Azure Dev Spaces** from the debug menu. 

    ![Azure Dev Space debugging](../media/demo-script/21-change-debug-target.png)

1. Hit F5 to start the debugger (or click the toolbar button in Visual Studio). 
    
    ![Site running](../media/demo-script/23-search-during-debug.png)

1. Once the site opens, zoom in on the URL to show the Dev Space prefix. 
1. Scroll down and search for **Seattle** in the city search box. 
1. In a moment, Visual Studio should obtain focus and the debugger should stop on the line with the breakpoint. 
    
    ![Search for Seattle](../media/demo-script/24-breakpoint-hit.png)

1. Hit F5 again to let the code continue running. At this point the API should return a match and the page should render the result on-screen. 
    
    ![Success](../media/demo-script/25-search-works.png)
