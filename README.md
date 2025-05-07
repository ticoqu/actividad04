# Actividad 4: Uso de variables para personalizar máquinas virtuales

## Descripción
Aprenderás a usar variables en Terraform para evitar valores estáticos y hacer más flexible la creación de máquinas.

### 📁 Archivos requeridos
- `main.tf`
- `cloud-config.yaml` (para entender qué hace cloud-init)

### 🎯 Objetivo
Parametrizar las máquinas virtuales mediante variables para mayor flexibilidad.

### Entrega
- Captura de pantalla del estado final de terraform apply.
- Archivos .tf.

## Pasos a seguir

1. **Crear `variables.tf`**
   ```hcl
   variable "cluster_psql" {
     type = list(object({
       name   = string
       ip     = string
       target = string
       type   = string
     }))
   }
   ```
2. **Modificar main.tf para usar variables**
   ```hcl
   resource "proxmox_vm_qemu" "cluster_psql" {
  for_each = { for index, vm in var.cluster_psql : vm.name => vm }

  name        = each.value.name
  desc        = "Servidores cluster psql"
  clone       = "debian-12-qcow2-template"

  cores       = each.value.type == "worker" ? 4 : 2
  sockets     = 1
  memory      = each.value.type == "worker" ? 4096 : 2048
  scsihw      = "virtio-scsi-pci"
  agent       = 1
  qemu_os     = "l26"
  ciuser      = "automatizacion"
  cipassword  = "$6$xRwIN4XsEB.mf0"
  searchdomain = "agetic.gob.bo"
  nameserver  = "8.8.8.8"
  sshkeys     = file("./keys/automatizacion.pub")
  ipconfig0   = "ip=${each.value.ip}/24,gw=192.168.1.1"

  disks {
    virtio {
      virtio0 {
        disk {
          storage = "vol1"
          size    = "8G"
        }
      }
    }

    dynamic "virtio1" {
      for_each = each.value.type == "worker" ? [1] : []
      content {
        disk {
          storage = "vol1"
          size    = "2G"
        }
      }
    }

    ide {
      ide2 {
        cloudinit {
          storage = "vol1"
        }
      }
    }
  }

  network {
    model  = "virtio"
    bridge = "vmbr0"
    mtu    = 0
  }
}
```

3. **Crear terraform.tfvars**
```hcl
cluster_psql = [
  { name = "master-psql", ip = "192.168.1.20", target = "pve", type = "master" },
  { name = "worker-psql1", ip = "192.168.1.21", target = "pve", type = "worker" },
  { name = "worker-psql2", ip = "192.168.1.22", target = "pve", type = "worker" }
]
```

4. **Verificar tareas**
   ```bash
   terraform plan
   ```
   
5. **Ejecutar el despliegue**
   ```bash
   terraform apply
   ```
