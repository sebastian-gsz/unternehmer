---
layout:
  width: default
  title:
    visible: false
  description:
    visible: false
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
  metadata:
    visible: true
  tags:
    visible: true
---

# Trabajar con múltiples VPCs

## Trabajar con múltiples VPCs

<figure><img src="../../../../../.gitbook/assets/Trabaja con varias redes de VPC.png" alt=""><figcaption></figcaption></figure>

***

{% stepper %}
{% step %}
<p align="center"><strong>Crear múltiples redes en modo personalizado con reglas de firewall</strong></p>

{% tabs %}
{% tab title="managementnet" %}
{% code overflow="wrap" %}
```bash
gcloud compute networks create managementsubnet --project="Ingresa el ID de tu proyecto" --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional --bgp-best-path-selection-mode=legacy && gcloud compute networks subnets create managementsubnet-us --project="Ingresa el ID de tu proyecto" --range=10.130.0.0/20 --stack-type=IPV4_ONLY --network=managementsubnet --region=us-east4
```
{% endcode %}
{% endtab %}

{% tab title="privatenet" %}
{% code overflow="wrap" %}
```bash
gcloud compute networks create privatenet --subnet-mode=custom && gcloud compute networks subnets create privatesubnet-us --network=privatenet --region=us-east4 --range=172.16.0.0/24 && gcloud compute networks subnets create privatesubnet-notus --network=privatenet --region=asia-east1 --range=172.20.0.0/20
```
{% endcode %}
{% endtab %}
{% endtabs %}

* Para ver una lista de las redes de VPC disponibles, ejecuta el siguiente comando: `gcloud compute networks list`. Para ver una lista de las subredes de VPC disponibles (ordenadas según la red de VPC), ejecuta el siguiente comando: `gcloud compute networks subnets list --sort-by=NETWORK`.

**Crea reglas de firewall para las nuevas subredes para permitir el tráfico de entrada SSH, ICMP y RDP.**&#x20;

{% tabs %}
{% tab title="managementnet" %}
{% code overflow="wrap" %}
```bash
gcloud compute --project="Ingresa el ID de tu proyecto" firewall-rules create managementnet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=managementsubnet --action=ALLOW --rules=tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
```
{% endcode %}
{% endtab %}

{% tab title="privatenet" %}
{% code overflow="wrap" %}
```bash
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
```
{% endcode %}
{% endtab %}
{% endtabs %}

* Para ver una lista de todas las reglas de firewall (ordenadas según la red de VPC), ejecuta el siguiente comando: `gcloud compute firewall-rules list --sort-by=NETWORK`

Una regla de firewall que creas se aplica a toda la red de VPC, independientemente de la región o de las subredes que contenga. Las reglas se implementan a nivel de las máquinas virtuales (VM), pero su configuración y alcance se definen en el contexto global de la VPC.
{% endstep %}

{% step %}
<p align="center"><strong>Crea nuevas instancias Virtual Machine (Compute Engine)</strong></p>

{% code overflow="wrap" %}
```bash
gcloud compute instances create managementnet-us-vm --project="Ingresa el ID de tu proyecto" --zone=us-east4-a --machine-type=e2-standard-4 --network-interface=network-tier=PREMIUM,stack-type=IPV4_ONLY,subnet=managementsubnet-us --metadata=enable-osconfig=TRUE,enable-oslogin=true --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=93375061552-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=managementnet-us-vm,image=projects/debian-cloud/global/images/debian-12-bookworm-v20250709,mode=rw,size=10,type=pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --labels=goog-ops-agent-policy=v2-x86-template-1-4-0,goog-ec-src=vm_add-gcloud --reservation-affinity=any && printf 'agentsRule:\n  packageState: installed\n  version: latest\ninstanceFilter:\n  inclusionLabels:\n  - labels:\n      goog-ops-agent-policy: v2-x86-template-1-4-0\n' > config.yaml && gcloud compute instances ops-agents policies create goog-ops-agent-v2-x86-template-1-4-0-us-east4-a --project="Ingresa el ID de tu proyecto" --zone=us-east4-a --file=config.yaml && gcloud compute resource-policies create snapshot-schedule default-schedule-1 --project="Ingresa el ID de tu proyecto" --region=us-east4 --max-retention-days=14 --on-source-disk-delete=keep-auto-snapshots --daily-schedule --start-time=07:00 && gcloud compute disks add-resource-policies managementnet-us-vm --project="Ingresa el ID de tu proyecto" --zone=us-east4-a --resource-policies=projects/"Ingresa el ID de tu proyecto"/regions/us-east4/resourcePolicies/default-schedule-1
```
{% endcode %}

<p align="center"><strong>Verifica y gestiona las instancias de VM creadas</strong></p>

```bash
gcloud compute instances list
```

The command `gcloud compute instances create privatenet-us-vm` is used to create a new virtual machine (VM) instance named `privatenet-us-vm` in Google Cloud's Compute Engine. The options specify the following configuration:

* `--zone=us-east4-a`: The instance will be created in the `us-east4-a` zone.
* `--machine-type=e2-medium`: The VM will use the `e2-medium` machine type, which typically includes 2 vCPUs and 4 GB of memory.
* `--subnet=privatesubnet-us`: The VM will be connected to the `privatesubnet-us` subnet within the configured virtual private cloud (VPC).

This setup allows for the VM to be part of the specified network and subnet, optimizing for the stipulated zone and machine type requirements.

{% code overflow="wrap" %}
```
gcloud compute instances create privatenet-us-vm --zone=us-east4-a --machine-type=e2-medium --subnet=privatesubnet-us
```
{% endcode %}

```hcl
# Este código es compatible con Terraform 4.25.0 y versiones compatibles con 4.25.0.
# Para obtener información sobre la validación de este código de Terraform, consulta https://developer.hashicorp.com/terraform/tutorials/gcp-get-started/google-cloud-platform-build#format-and-validate-the-configuration

resource "google_compute_instance" "managementnet-us-vm" {
  boot_disk {
    auto_delete = true
    device_name = "managementnet-us-vm"

    initialize_params {
      image = "projects/debian-cloud/global/images/debian-12-bookworm-v20250709"
      size  = 10
      type  = "pd-balanced"
    }

    mode = "READ_WRITE"
  }

  can_ip_forward      = false
  deletion_protection = false
  enable_display      = false

  labels = {
    goog-ec-src           = "vm_add-tf"
    goog-ops-agent-policy = "v2-x86-template-1-4-0"
  }

  machine_type = "e2-standard-4"

  metadata = {
    enable-osconfig = "TRUE"
    enable-oslogin  = "true"
  }

  name = "managementnet-us-vm"

  network_interface {
    access_config {
      network_tier = "PREMIUM"
    }

    queue_count = 0
    stack_type  = "IPV4_ONLY"
    subnetwork  = "projects/qwiklabs-gcp-03-5c13adb7889d/regions/us-east4/subnetworks/managementsubnet-us"
  }

  scheduling {
    automatic_restart   = true
    on_host_maintenance = "MIGRATE"
    preemptible         = false
    provisioning_model  = "STANDARD"
  }

  service_account {
    email  = "93375061552-compute@developer.gserviceaccount.com"
    scopes = ["https://www.googleapis.com/auth/devstorage.read_only", "https://www.googleapis.com/auth/logging.write", "https://www.googleapis.com/auth/monitoring.write", "https://www.googleapis.com/auth/service.management.readonly", "https://www.googleapis.com/auth/servicecontrol", "https://www.googleapis.com/auth/trace.append"]
  }

  shielded_instance_config {
    enable_integrity_monitoring = true
    enable_secure_boot          = false
    enable_vtpm                 = true
  }

  zone = "us-east4-a"
}

module "ops_agent_policy" {
  source          = "github.com/terraform-google-modules/terraform-google-cloud-operations/modules/ops-agent-policy"
  project         = "qwiklabs-gcp-03-5c13adb7889d"
  zone            = "us-east4-a"
  assignment_id   = "goog-ops-agent-v2-x86-template-1-4-0-us-east4-a"
  agents_rule = {
    package_state = "installed"
    version = "latest"
  }
  instance_filter = {
    all = false
    inclusion_labels = [{
      labels = {
        goog-ops-agent-policy = "v2-x86-template-1-4-0"
      }
    }]
  }
}

```
{% endstep %}
{% endstepper %}
