# Getting Started: End to End Tests

The purpose of End to End "e2e" Tests is to make sure the behaviour of the software, infrastructure, etc., is what we expect. With TF2, you can implement e2e tests to test your Terraform codes after they are deployed to real infrastructures.

In the case of Terraform:

  - DevOps engineers are responsible for writing Terraform codes as well as writing e2e tests and integrating them into CI/CD pipelines.

From now on, TF2 is always there for you to test your Terraform-provisioned infrastructures. Not only infrastructures but also everything you managed and deployed by the Terraform, as well as all Terraform <u>data</u> resources. Moreover, TF2 can be integrated into your CD pipelines, and after deploying the new Terraform codes, you can run the tests to test the infrastructure's behaviour in the real world. If something goes wrong or is unusual, you can detect it and rollback the system to the previous working state. With this approach, you can deliver better and more reliable infrastructures and reduce your unexpected situations and downtimes.

!!! check "I'm compatible"
    TF2 is fully compatible with Jenkins CI, Gitlab CI, GitHub Actions and Azure Pipelines.

## Your first End to End test

Consider the following Terraform code. It's a standard Terraform code to deploy a Docker container. To test this scenario, you should install Docker on your local machine.

```tf title="main.tf"
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = ">= 2.16"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_container" "nginx" {
  name    = "nginx"
  image   = "nginx:1"
  restart = "unless-stopped"
}

output "nginx_address" {
  value = "http://localhost"
}
```

Now deploy the Docker container by running the following commands:

<div id="termynal-terraform-deploy-docker-container" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">terraform init</span>
  <span data-ty="progress" data-ty-progressChar="="></span>
  <span data-ty="input" data-ty-prompt="$">terraform apply</span>
  <span data-ty>Do you want to perform these actions?</span>
  <span data-ty="input" data-ty-prompt="Enter a value:">yes</span>
  <span data-ty>docker_container.nginx: Creating...</span>
  <span data-ty="progress" data-ty-progressChar="="></span>
  <span data-ty>Apply complete! Resources: 1 added, 0 destroyed.</span>
  <span data-ty>Outputs:</span>
  <span data-ty>nginx_address = "http://localhost"</span>
</div>

Our desired state about the deployed container:

  - The container should be run without any problem.
  - The container can be accessed by the <u>nginx_address</u> output.

To automate testing the container status and accessibility, write the following code:

```py title="tests.py"
from tf2 import Tf2, LocalCommandExecutor

tf2 = Tf2()

@tf2.test("resources.docker_container.nginx")
def test_nginx_status(self):
    assert LocalCommandExecutor(
        f"docker container inspect { self.values.name } -f '{{{{ .State.Status }}}}'"
    ).result.stdout.strip() == "running"

@tf2.test("resources.docker_container.nginx")
def test_nginx_public_access(self):
    assert LocalCommandExecutor(f"curl -s { tf2.outputs.nginx_address.value }").result.rc == 0

tf2.run()
```

Now, run the test script and see the result. Before running tests, make sure you have installed the **tf2project** package. Run `pip install tf2project` to install it on your machine.

Run your TF2 tests with the following command:

<div id="termynal-tf2-run-tests" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">python tests.py</span>
</div>

<p align="center">
  <img src="/assets/img/getting-started-end-to-end-tests-failed-result.png" alt="TF2 Tests Failed">
</p>

As you can see, the test has failed because the container cannot be accessed through the network as we expected. Actually, this problem happened because of resource misconfiguration, and such problems are out of Terraform's scope, and it cannot detect them.

Now update the Terraform code to the following code, apply it, and run the tests again.

```tf title="main.tf" hl_lines="18-21"
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = ">= 2.16"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

resource "docker_container" "nginx" {
  name    = "nginx"
  image   = "nginx:1"
  restart = "unless-stopped"
  ports {
    internal = 80
    external = 80
  }
}

output "nginx_address" {
  value = "http://localhost"
}
```

Run the following commands to apply and test:

<div id="termynal-terraform-redeploy-docker-container" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">terraform apply</span>
  <span data-ty>Do you want to perform these actions?</span>
  <span data-ty="input" data-ty-prompt="Enter a value:">yes</span>
  <span data-ty>docker_container.nginx: Destroying...</span>
  <span data-ty>docker_container.nginx: Destruction complete after 0s</span>
  <span data-ty>docker_container.nginx: Creating...</span>
  <span data-ty="progress" data-ty-progressChar="="></span>
  <span data-ty>Apply complete! Resources: 1 added, 1 destroyed.</span>
  <span data-ty>Outputs:</span>
  <span data-ty>nginx_address = "http://localhost"</span>
  <span data-ty="input" data-ty-prompt="$">python tests.py</span>
</div>

<p align="center">
  <img src="/assets/img/getting-started-end-to-end-tests-passed-result.png" alt="TF2 Tests Passed">
</p>

!!! info "Result"
    If tests pass, the test script will be exited with return code **0**, and **passed** will be written to the <u>.tf2result</u> file, and if they fail, the test script will be exited with return code **1**, and **failed** will be written to the <u>.tf2result</u> file.

That's it. You wrote your first End to End test with Terraform Test Framework. Now, you can deliver better and more reliable Terraform-provisioned infrastructures.

*[e2e]: End-to-End
*[CI]: Continuous Integration
*[CD]: Continuous Deployment
