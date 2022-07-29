# Getting Started: Policy as Code

Policy as Code, **PaC**, is a set of rules and conditions which is written as a code.

With Policy-as-Code, you can implement compliance tests and enforce developers to write their codes in a standard way which fits your company rules.

In the case of Terraform:

  - DevOps engineers write Terraform codes.
  - You write Policy-as-Code codes.

When DevOps engineers make any changes in repositories containing Terraform codes, the PaC codes will be run as part of CI pipelines. If they have not followed PaC rules, the pipeline fails, and changes are not accepted to be merged into the main branch.

!!! check "I'm compatible"
    TF2 is fully compatible with Jenkins CI, Gitlab CI, GitHub Actions and Azure Pipelines.

## Your first Policy as Code

Consider the following Terraform code. This is a standard Terraform code which is written by DevOps engineers to deploy a Deployment Resource in a Kubernetes environment. Please note that to test it, you should have a working Kubernetes cluster.

```tf title="main.tf"
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_deployment" "nginx_deployment" {
  metadata {
    name = "nginx"
    labels = {
      "app.kubernetes.io/name"       = "nginx"
      "app.kubernetes.io/created-by" = "tf2project"
    }
  }
  spec {
    replicas = 1
    selector {
      match_labels = {
        "app.kubernetes.io/name"       = "nginx"
        "app.kubernetes.io/created-by" = "tf2project"
      }
    }
    template {
      metadata {
        labels = {
          "app.kubernetes.io/name"       = "nginx"
          "app.kubernetes.io/created-by" = "tf2project"
        }
      }
      spec {
        container {
          name  = "nginx"
          image = "nginx:latest"
          port {
            container_port = 80
          }
        }
      }
    }
  }
}
```

Now, it's time to implement your **Policy as Code** to enforce some rules.

With the following code, these policies will be applied to the **nginx_deployment** resource:

  - [x] The resource should have the <u>namespace</u> argument, and its value should be anything except <u>default</u> to avoid running the resource in the default namespace.
  - [x] The resource should have the <u>app.kubernetes.io/env</u> label, and its value should be <u>development</u> or <u>production</u> to pass this policy rule.
  - [x] The resource should have at least <u>2</u> replicas. Due to the use of the <u>ignore_errors</u> argument, it's an optional rule and can be ignored by the engine.
  - [x] The resource should have the image argument with a specific Nginx version and cannot use the <u>latest</u> tag of the docker image.

```py title="tests.py"
from tf2 import Tf2, Terraform, TerraformPlanLoader

tf2 = Tf2(Terraform(TerraformPlanLoader()))

@tf2.test("resources.kubernetes_deployment.nginx_deployment")
def test_nginx_namespace_not_default(self):
    assert self.values.metadata[0].namespace != "default"

@tf2.test("resources.kubernetes_deployment.nginx_deployment")
def test_nginx_label_env(self):
    assert (self.values.metadata[0].labels.app_kubernetes_io_env in ["production", "development"]) is True
    assert (self.values.spec[0].template[0].metadata[0].labels.app_kubernetes_io_env in ["production", "development"]) is True

@tf2.test("resources.kubernetes_deployment.nginx_deployment", ignore_errors=True)
def test_nginx_min_replicas(self):
    assert int(self.values.spec[0].replicas) >= 2

@tf2.test("resources.kubernetes_deployment.nginx_deployment")
def test_nginx_image_not_latest(self):
    assert self.values.spec[0].template[0].spec[0].container[0].image.count(":") == 1
    assert self.values.spec[0].template[0].spec[0].container[0].image.endswith(":latest") is False

tf2.run()
```

Now, create the Terraform plan file:

<div id="termynal-terraform" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">terraform init</span>
  <span data-ty="progress" data-ty-progressChar="="></span>
  <span data-ty="input" data-ty-prompt="$">terraform plan -out terraform.tfplan</span>
  <span data-ty>Saved the plan to: terraform.tfplan</span>
  <span data-ty="input" data-ty-prompt="$">ls</span>
  <span data-ty>main.tf  providers.tf  terraform.tfplan  tests.py</span>
</div>

!!! important
    If you use any CI/CD systems to automate your Terraform pipelines, save the **terraform.tfplan** file as an artifact of the pipeline. In the case of any errors, you can investigate them by using this file.

It's time to put everything together and run the test. Before running tests, make sure you have installed the **tf2project** package. Run `pip install tf2project` to install it.

Run your TF2 tests with the following command:

<div id="termynal-tf2-run-tests" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">python tests.py</span>
</div>

<p align="center">
  <img src="/assets/img/getting-started-policy-as-code-failed-result.png" alt="TF2 Tests Failed">
</p>

The test failed because the Terraform code does not follow our policies.

Now, update the Terraform code to the following code to follow the policies, recreate the Terraform plan and run **tests.py** again to see the result.

```tf title="main.tf" hl_lines="8 12 16 21 29 35"
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_deployment" "nginx_deployment" {
  metadata {
    name      = "nginx"
    namespace = "test"
    labels = {
      "app.kubernetes.io/name"       = "nginx"
      "app.kubernetes.io/created-by" = "tf2project"
      "app.kubernetes.io/env"        = "development"
    }
  }
  spec {
    replicas = 3
    selector {
      match_labels = {
        "app.kubernetes.io/name"       = "nginx"
        "app.kubernetes.io/created-by" = "tf2project"
        "app.kubernetes.io/env"        = "development"
      }
    }
    template {
      metadata {
        labels = {
          "app.kubernetes.io/name"       = "nginx"
          "app.kubernetes.io/created-by" = "tf2project"
          "app.kubernetes.io/env"        = "development"
        }
      }
      spec {
        container {
          name  = "nginx"
          image = "nginx:1"
          port {
            container_port = 80
          }
        }
      }
    }
  }
}
```

Create the Terraform plan file:

<div id="termynal-terraform-plan-file" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">terraform plan -out terraform.tfplan</span>
</div>

Run TF2 tests with the following command:

<div id="termynal-tf2-run-tests" data-termynal data-ty-typeDelay="40" data-ty-lineDelay="700">
  <span data-ty="input" data-ty-prompt="$">python tests.py</span>
</div>

<p align="center">
  <img src="/assets/img/getting-started-policy-as-code-passed-result.png" alt="TF2 Tests Passed">
</p>

!!! info "Result"
    If tests pass, the test script will be exited with return code **0**, and **passed** will be written to the <u>.tf2result</u> file, and if they fail, the test script will be exited with return code **1**, and **failed** will be written to the <u>.tf2result</u> file.

That's it. You wrote your first Policy as Code with Terraform Test Framework. Now, you can deliver more reliable and secure Terraform codes.

*[PaC]: Policy as Code
*[CI]: Continuous Integration
*[CD]: Continuous Deployment
