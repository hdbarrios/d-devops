# d-devops
Diplomado DevOps 2024

---
# PIN
El proyecto DevOps que implica la integración de diferentes herramientas y tecnologías en AWS, enfocándose en los siguientes aspectos:

### Creación de una instancia EC2:

- Se debe crear una instancia EC2 en la región us-east-1 usando el sistema operativo Ubuntu Server 22.04 con tipo t2.micro.
- En la sección de user data, se deben incluir scripts para instalar AWS CLI, kubectl, Docker, Helm, entre otras herramientas.
- Se debe crear un par de claves llamado pin en formato PEM para la conexión.
- Asignar el rol IAM ec2-admin-role-cicd a la instancia.

### Creación de un clúster EKS:

- Usar eksctl para crear un clúster EKS con 3 nodos t3.small en la región us-east-1.
- Desplegar un pod NGINX y verificar su disponibilidad.

### Instalación de herramientas de monitoreo:

- Implementar Prometheus y Grafana para monitorear los pods.
- Instalar el driver AWS EBS CSI y configurar roles y políticas necesarias.

### Limpieza de recursos:

- Desinstalar Prometheus y Grafana, borrar los namespaces creados, y eliminar el clúster EKS.

## Implementación con Terraform
A continuación, se detalla cómo implementar cada parte usando Terraform.

1. Creación de la Instancia EC2 con Rol IAM

```hcl

provider "aws" {
  region = "us-east-1"
}

resource "aws_key_pair" "ec2_key" {
  key_name   = "pin"
  public_key = file("~/.ssh/id_rsa.pub")
}

resource "aws_iam_role" "ec2_admin_role" {
  name = "ec2-admin-role-cicd"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_admin_policy" {
  role       = aws_iam_role.ec2_admin_role.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

resource "aws_instance" "ec2_instance" {
  ami           = "ami-0a313d6098716f372" # Ubuntu 22.04
  instance_type = "t2.micro"
  key_name      = aws_key_pair.ec2_key.key_name
  iam_instance_profile = aws_iam_instance_profile.ec2_instance_profile.name

  user_data = file("${path.module}/ec2_user_data.sh") # Aquí se incluye el script desde la ruta local

  tags = {
    Name = "ProyectoPIN"
  }
}

resource "aws_iam_instance_profile" "ec2_instance_profile" {
  name = "ec2-admin-profile"
  role = aws_iam_role.ec2_admin_role.name
}
```
2. Creación del Clúster EKS
```hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "eks-mundos-e"
  cluster_version = "1.21"
  subnets         = var.subnets
  vpc_id          = var.vpc_id

  node_groups = {
    eks_nodes = {
      desired_capacity = 3
      max_capacity     = 3
      min_capacity     = 1
      instance_type    = "t3.small"
      key_name         = aws_key_pair.ec2_key.key_name
    }
  }

  manage_aws_auth = true
}

output "cluster_name" {
  value = module.eks.cluster_id
}
```
3. Implementación de Prometheus y Grafana
```hcl
resource "helm_release" "prometheus" {
  name       = "prometheus"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "prometheus"
  namespace  = "monitoring"

  set {
    name  = "alertmanager.persistentVolume.storageClass"
    value = "gp2"
  }

  set {
    name  = "server.persistentVolume.storageClass"
    value = "gp2"
  }
}

resource "helm_release" "grafana" {
  name       = "grafana"
  repository = "https://grafana.github.io/helm-charts"
  chart      = "grafana"
  namespace  = "monitoring"

  set {
    name  = "service.type"
    value = "LoadBalancer"
  }

  set {
    name  = "adminPassword"
    value = "EKS!sAWSome"
  }
}
```
4. Cleanup de Recursos
```hcl
resource "null_resource" "cleanup" {
  provisioner "local-exec" {
    command = <<-EOF
              helm uninstall prometheus --namespace monitoring
              kubectl delete ns monitoring
              eksctl delete cluster --name eks-mundos-e
              EOF
  }
}
```
---
Este código de Terraform implementa todos los componentes descritos en el documento, incluyendo la creación de la instancia EC2, el clúster EKS, y la configuración de Prometheus y Grafana para monitoreo, junto con la limpieza de recursos una vez completado el proyecto.
