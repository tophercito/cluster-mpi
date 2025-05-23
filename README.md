# Guía Completa: Cluster Beowulf MPI
## Fedora 41 (Maestro) + Ubuntu 24.04 (Esclavos)

### Requisitos Previos
- **Nodo Maestro:** Fedora Linux 41 (Workstation Edition)
- **Nodos Esclavos:** Ubuntu 24.04.2 LTS
- Red cableada con conectividad entre todos los nodos
- Acceso administrativo (sudo) en todos los nodos

### Script de Reparación Automática

Crear `/home/cluster/shared/reparar_mpi.sh`:

```bash
#!/bin/bash

echo "=== SCRIPT DE REPARACIÓN MPI ==="
echo "Este script detecta y repara problemas de versiones MPI"
echo ""

NODOS=("master-node" "slave-node1" "slave-node2")
VERSION_ESPERADA="4.1.6"
PROBLEMAS_ENCONTRADOS=0

# Verificar problemas
for node in "${NODOS[@]}"; do
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        version=$(ssh cluster@$node "mpirun --version 2>/dev/null | head -1 | grep -o '[0-9]\+\.[0-9]\+\.[0-9]\+'" 2>/dev/null)
        if [ "$version" != "$VERSION_ESPERADA" ]; then
            echo "❌ $node tiene MPI versión $version (esperado $VERSION_ESPERADA)"
            PROBLEMAS_ENCONTRADOS=1
        fi
    fi
done

if [ $PROBLEMAS_ENCONTRADOS -eq 0 ]; then
    echo "✅ No se detectaron problemas de versiones MPI"
    exit 0
fi

echo ""
echo "Se detectaron problemas. ¿Deseas reinstalar OpenMPI 4.1.6 en todos los nodos? (y/N)"
read -r respuesta

if [[ "$respuesta" =~ ^[Yy]$ ]]; then
    echo "Reinstalando OpenMPI en todos los nodos..."
    
    for node in "${NODOS[@]}"; do
        echo "Procesando $node..."
        ssh cluster@$node "
            # Limpiar instalaciones previas del gestor de paquetes
            if command -v dnf &> /dev/null; then
                sudo dnf remove -y openmpi openmpi-devel 2>/dev/null || true
            elif command -v apt &> /dev/null; then
                sudo apt remove -y openmpi-bin openmpi-common libopenmpi-dev 2>/dev/null || true
            fi
            
            # Remover OpenMPI compilado si existe
            sudo rm -rf /opt/openmpi-4.1.6 /opt/openmpi 2>/dev/null || true
            
            # Limpiar variables de entorno
            sed -i '/# OpenMPI Environment/,+4d' ~/.bashrc
            
            echo 'Nodo $node limpiado'
        "
    done
    
    echo ""
    echo "✅ Limpieza completada. Ahora ejecuta manualmente la Fase 2.5 en cada nodo."
    echo "Comando rápido para cada nodo:"
    echo ""
    echo "cd ~ && wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.6.tar.gz"
    echo "tar -xzf openmpi-4.1.6.tar.gz && cd openmpi-4.1.6"
    echo "./configure --prefix=/opt/openmpi-4.1.6 --enable-mpi-cxx --enable-mpi-fortran --with-slurm=no --enable-shared"
    echo "make -j\$(nproc) && sudo make install"
    echo "sudo ln -sf /opt/openmpi-4.1.6 /opt/openmpi"
else
    echo "Operación cancelada"
fi
```

Hacer ejecutables los scripts:
```bash
chmod +x /home/cluster/shared/verificar_versiones.sh
chmod +x /home/cluster/shared/reparar_mpi.sh
```

---

## 🔧 FASE 2.5: Instalación Unificada de OpenMPI (TODOS los nodos)

### ⚠️ IMPORTANTE: Esta fase debe ejecutarse en TODOS los nodos (maestro y esclavos)

```bash
# Descargar OpenMPI 4.1.6 (versión estable y compatible)
cd ~/mpi_build
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.6.tar.gz

# Verificar descarga
ls -la openmpi-4.1.6.tar.gz

# Extraer
tar -xzf openmpi-4.1.6.tar.gz
cd openmpi-4.1.6

# Configurar compilación
./configure --prefix=/opt/openmpi-4.1.6 \
            --enable-mpi-cxx \
            --enable-mpi-fortran \
            --with-slurm=no \
            --enable-shared

# Compilar (esto toma 10-20 minutos)
make -j$(nproc)

# Instalar
sudo make install

# Crear enlace simbólico para facilidad
sudo ln -sf /opt/openmpi-4.1.6 /opt/openmpi

# Configurar variables de entorno
cat >> ~/.bashrc << 'EOF'
# OpenMPI Environment
export MPI_HOME=/opt/openmpi
export PATH=$MPI_HOME/bin:$PATH
export LD_LIBRARY_PATH=$MPI_HOME/lib:$LD_LIBRARY_PATH
export MANPATH=$MPI_HOME/share/man:$MANPATH
EOF

# Aplicar configuración
source ~/.bashrc

# Verificar instalación
mpirun --version
mpicc --version

# Limpiar archivos temporales
cd ~
rm -rf ~/mpi_build
```

### Verificación de Versión Unificada

**Ejecutar en TODOS los nodos para confirmar la misma versión:**

```bash
echo "=== Verificando OpenMPI en $(hostname) ==="
echo "Versión MPI:"
mpirun --version | head -1

echo "Ubicación binarios:"
which mpirun
which mpicc

echo "Variables de entorno:"
echo "MPI_HOME: $MPI_HOME"
echo "PATH contiene: $(echo $PATH | grep -o '/opt/openmpi[^:]*')"
```

**Resultado esperado en TODOS los nodos:**
```
=== Verificando OpenMPI en [hostname] ===
Versión MPI:
mpirun (Open MPI) 4.1.6

Ubicación binarios:
/opt/openmpi/bin/mpirun
/opt/openmpi/bin/mpicc

Variables de entorno:
MPI_HOME: /opt/openmpi
PATH contiene: /opt/openmpi/bin
```

---

## 🔧 FASE 1: Configuración del Nodo Maestro (Fedora 41)

### 1.1 Crear Usuario Cluster
```bash
# Crear usuario
sudo useradd -m -s /bin/bash cluster
sudo passwd cluster
# Contraseña: cluster123

# Agregar al grupo wheel para sudo
sudo usermod -aG wheel cluster

# Cambiar a usuario cluster
su - cluster
```

### 1.2 Configurar SSH
```bash
# Instalar SSH cliente
sudo dnf install -y openssh-clients openssh-server

# Habilitar y iniciar SSH
sudo systemctl enable sshd
sudo systemctl start sshd

# Generar claves SSH
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Configurar SSH sin contraseña para localhost
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### 1.3 Instalar Dependencias para Compilar MPI
```bash
# Instalar herramientas de compilación
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y gcc gcc-c++ gcc-gfortran wget

# Crear directorio de trabajo
mkdir -p ~/mpi_build
cd ~/mpi_build
```

### 1.4 Configurar NFS (Sistema de Archivos Compartido)
```bash
# Instalar NFS
sudo dnf install -y nfs-utils

# Crear directorio compartido
mkdir -p /home/cluster/shared
chmod 755 /home/cluster/shared

# Configurar exports
sudo bash -c 'echo "/home/cluster/shared *(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports'

# Iniciar servicios NFS
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
sudo exportfs -ra
```

### 1.5 Configurar Firewall
```bash
# Permitir SSH y NFS
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

---

## 🔧 FASE 2: Configuración de Nodos Esclavos (Ubuntu 24.04)

### 2.1 Crear Usuario Cluster
```bash
# Crear usuario
sudo useradd -m -s /bin/bash cluster
sudo passwd cluster
# Contraseña: cluster123

# Agregar al grupo sudo
sudo usermod -aG sudo cluster

# Cambiar a usuario cluster
su - cluster
```

### 2.2 Configurar SSH
```bash
# Instalar SSH
sudo apt update
sudo apt install -y openssh-server

# Habilitar SSH
sudo systemctl enable ssh
sudo systemctl start ssh

# Crear directorio SSH
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

### 2.3 Instalar Dependencias para Compilar MPI
```bash
# Actualizar sistema e instalar herramientas de compilación
sudo apt update
sudo apt install -y build-essential gcc g++ gfortran wget

# Crear directorio de trabajo
mkdir -p ~/mpi_build
cd ~/mpi_build
```

### 2.4 Configurar NFS Cliente
```bash
# Instalar cliente NFS
sudo apt install -y nfs-common

# Crear punto de montaje
mkdir -p /home/cluster/shared
```

---

## 🔧 FASE 3: Configuración de Red y Conectividad

### 3.1 Configurar /etc/hosts (En TODOS los nodos)

Obtener IPs de todos los nodos:
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

Editar `/etc/hosts` en todos los nodos:
```bash
sudo nano /etc/hosts
```

Agregar (ajustar IPs según tu red):
```
# Cluster Beowulf
192.168.1.100    master-node
192.168.1.101    slave-node1
192.168.1.102    slave-node2
# Agregar más nodos según necesites
```

### 3.2 Configurar SSH sin Contraseña (Desde el maestro)

```bash
# En el nodo maestro, copiar clave pública a cada esclavo
ssh-copy-id cluster@slave-node1
ssh-copy-id cluster@slave-node2
# Repetir para cada nodo esclavo

# Probar conexión sin contraseña
ssh cluster@slave-node1 "hostname"
```

### 3.3 Montar NFS en Esclavos

En cada nodo esclavo:
```bash
# Montar directorio compartido
sudo mount -t nfs master-node:/home/cluster/shared /home/cluster/shared

# Configurar montaje automático
sudo bash -c 'echo "master-node:/home/cluster/shared /home/cluster/shared nfs defaults 0 0" >> /etc/fstab'
```

---

## 🔧 FASE 4: Configuración del Cluster

### 4.1 Crear Archivo de Hosts MPI (En el maestro)

```bash
# Crear archivo hostfile
cat > /home/cluster/shared/hostfile << EOF
master-node slots=4
slave-node1 slots=4
slave-node2 slots=4
EOF
```

### 4.2 Probar Conectividad MPI

```bash
# Desde el nodo maestro
cd /home/cluster/shared

# Probar comando simple en todos los nodos
mpirun -hostfile hostfile -np 8 hostname

# Probar programa básico
mpirun -hostfile hostfile -np 8 /bin/echo "Hola desde el nodo: $(hostname)"
```

---

## 💻 FASE 5: Programa de Números Perfectos Mejorado

### 5.1 Código C Optimizado

Crear archivo `/home/cluster/shared/numeros_perfectos.c`:

```c
#include <mpi.h>
#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <time.h>

int es_perfecto(long long n) {
    if (n <= 1) return 0;
    
    long long suma = 1;
    long long limite = (long long)sqrt(n);
    
    for (long long i = 2; i <= limite; i++) {
        if (n % i == 0) {
            suma += i;
            if (i != n / i) {
                suma += n / i;
            }
        }
    }
    
    return suma == n;
}

int main(int argc, char *argv[]) {
    int rank, size;
    long long inicio, fin, rango_total;
    long long inicio_local, fin_local, rango_local;
    long long perfectos_locales = 0, perfectos_totales = 0;
    double tiempo_inicio, tiempo_fin;
    
    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    
    if (argc != 3) {
        if (rank == 0) {
            printf("Uso: %s <inicio> <fin>\n", argv[0]);
        }
        MPI_Finalize();
        return 1;
    }
    
    inicio = atoll(argv[1]);
    fin = atoll(argv[2]);
    rango_total = fin - inicio + 1;
    rango_local = rango_total / size;
    
    // Calcular rango para cada proceso
    inicio_local = inicio + rank * rango_local;
    fin_local = inicio_local + rango_local - 1;
    
    // El último proceso toma los números restantes
    if (rank == size - 1) {
        fin_local = fin;
    }
    
    if (rank == 0) {
        printf("=== CLUSTER BEOWULF MPI - NÚMEROS PERFECTOS ===\n");
        printf("Rango total: %lld - %lld\n", inicio, fin);
        printf("Procesos: %d\n", size);
        printf("============================================\n");
    }
    
    tiempo_inicio = MPI_Wtime();
    
    printf("Proceso %d: analizando rango %lld - %lld\n", rank, inicio_local, fin_local);
    
    // Buscar números perfectos en el rango local
    for (long long n = inicio_local; n <= fin_local; n++) {
        if (es_perfecto(n)) {
            perfectos_locales++;
            printf("Proceso %d encontró número perfecto: %lld\n", rank, n);
        }
        
        // Mostrar progreso cada 100000 números
        if ((n - inicio_local) % 100000 == 0 && rank == 0) {
            double progreso = (double)(n - inicio) / rango_total * 100;
            printf("Progreso: %.2f%%\n", progreso);
        }
    }
    
    // Reducir resultados
    MPI_Reduce(&perfectos_locales, &perfectos_totales, 1, MPI_LONG_LONG, MPI_SUM, 0, MPI_COMM_WORLD);
    
    tiempo_fin = MPI_Wtime();
    
    if (rank == 0) {
        printf("\n=== RESULTADOS ===\n");
        printf("Números perfectos encontrados: %lld\n", perfectos_totales);
        printf("Tiempo total: %.6f segundos\n", tiempo_fin - tiempo_inicio);
        printf("Rendimiento: %.2f números/segundo\n", rango_total / (tiempo_fin - tiempo_inicio));
        printf("==================\n");
    }
    
    MPI_Finalize();
    return 0;
}
```

### 5.2 Script de Compilación y Ejecución

Crear archivo `/home/cluster/shared/run_cluster.sh`:

```bash
#!/bin/bash

# Script para compilar y ejecutar en el cluster

PROGRAMA="numeros_perfectos"
HOSTFILE="/home/cluster/shared/hostfile"

echo "=== COMPILANDO PROGRAMA ==="
mpicc -o $PROGRAMA ${PROGRAMA}.c -lm -O3

if [ $? -eq 0 ]; then
    echo "Compilación exitosa"
else
    echo "Error en la compilación"
    exit 1
fi

echo ""
echo "=== EJECUTANDO PRUEBAS ==="

# Pruebas con diferentes configuraciones
CONFIGURACIONES=(
    "1:1-10000"
    "2:1-50000" 
    "4:1-100000"
    "8:1-500000"
)

for config in "${CONFIGURACIONES[@]}"; do
    IFS=':' read -r procesos rango <<< "$config"
    IFS='-' read -r inicio fin <<< "$rango"
    
    echo ""
    echo "--- Prueba con $procesos procesos, rango $inicio-$fin ---"
    
    time mpirun -hostfile $HOSTFILE -np $procesos ./$PROGRAMA $inicio $fin
    
    echo "Presiona Enter para continuar..."
    read
done

echo ""
echo "=== PRUEBAS COMPLETADAS ==="
```

### 5.3 Compilar y Ejecutar

```bash
# Hacer ejecutable el script
chmod +x /home/cluster/shared/run_cluster.sh

# Ejecutar pruebas
cd /home/cluster/shared
./run_cluster.sh
```

---

## 🧪 FASE 6: Verificación y Pruebas

### 6.1 Pruebas Básicas de Conectividad

```bash
# Verificar que todos los nodos responden
for node in master-node slave-node1 slave-node2; do
    echo "Probando $node..."
    ssh cluster@$node "echo 'Conectado a $(hostname)'"
done
```

### 6.2 Pruebas de Rendimiento MPI

```bash
# Benchmark básico
mpirun -hostfile /home/cluster/shared/hostfile -np 8 \
    /usr/lib64/openmpi/bin/osu_latency

# Prueba de ancho de banda
mpirun -hostfile /home/cluster/shared/hostfile -np 8 \
    /usr/lib64/openmpi/bin/osu_bw
```

### 6.3 Monitoreo del Cluster

Crear `/home/cluster/shared/verificar_versiones.sh`:

```bash
#!/bin/bash

echo "=== VERIFICACIÓN DE VERSIONES MPI ==="
echo "Fecha: $(date)"
echo ""

NODOS=("master-node" "slave-node1" "slave-node2")
VERSION_ESPERADA="4.1.6"

echo "--- Verificando OpenMPI en todos los nodos ---"
for node in "${NODOS[@]}"; do
    echo -n "$node: "
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        version=$(ssh cluster@$node "mpirun --version 2>/dev/null | head -1 | grep -o '[0-9]\+\.[0-9]\+\.[0-9]\+'" 2>/dev/null)
        if [ "$version" = "$VERSION_ESPERADA" ]; then
            echo "✅ OpenMPI $version (CORRECTO)"
        elif [ -n "$version" ]; then
            echo "❌ OpenMPI $version (INCORRECTO - esperado $VERSION_ESPERADA)"
        else
            echo "❌ MPI no encontrado o no funcional"
        fi
    else
        echo "❌ NODO INACCESIBLE"
    fi
done

echo ""
echo "--- Verificando rutas MPI ---"
for node in "${NODOS[@]}"; do
    echo "$node:"
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        mpi_path=$(ssh cluster@$node "which mpirun 2>/dev/null")
        if [[ "$mpi_path" == "/opt/openmpi/bin/mpirun" ]]; then
            echo "  ✅ Ruta correcta: $mpi_path"
        else
            echo "  ❌ Ruta incorrecta: $mpi_path (esperado: /opt/openmpi/bin/mpirun)"
        fi
    fi
done

echo ""
echo "--- Prueba de comunicación MPI ---"
cd /home/cluster/shared
if mpirun -hostfile hostfile -np 4 /opt/openmpi/bin/mpirun --version &>/dev/null; then
    echo "✅ Comunicación MPI funcional entre nodos"
else
    echo "❌ Error en comunicación MPI"
fi
```

Crear `/home/cluster/shared/monitor_cluster.sh`:

```bash
#!/bin/bash

echo "=== ESTADO DEL CLUSTER ==="
echo "Fecha: $(date)"
echo ""

echo "--- Nodos Activos ---"
for node in master-node slave-node1 slave-node2; do
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        load=$(ssh cluster@$node "uptime | awk -F'load average:' '{print \$2}'")
        echo "$node: ACTIVO - Carga:$load"
    else
        echo "$node: INACTIVO"
    fi
done

echo ""
echo "--- Uso de CPU por Nodo ---"
for node in master-node slave-node1 slave-node2; do
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        cpu=$(ssh cluster@$node "top -bn1 | grep 'Cpu(s)' | awk '{print \$2}' | cut -d'%' -f1")
        echo "$node: ${cpu}% CPU usado"
    fi
done

echo ""
echo "--- Memoria por Nodo ---"
for node in master-node slave-node1 slave-node2; do
    if ssh -o ConnectTimeout=5 cluster@$node "echo OK" &>/dev/null; then
        mem=$(ssh cluster@$node "free -m | awk 'NR==2{printf \"%.1f%%\", \$3*100/\$2 }'")
        echo "$node: $mem memoria usada"
    fi
done
```

---

## 🔍 Solución de Problemas Comunes

### ⚠️ Problema Principal: Versiones Incompatibles de MPI

**Síntoma:** Fedora 41 instala OpenMPI 5.x, Ubuntu 24.04 instala OpenMPI 4.x
```bash
# En Fedora: mpirun --version
# Resultado: Open MPI 5.0.x

# En Ubuntu: mpirun --version  
# Resultado: Open MPI 4.1.x
```

**Solución Implementada:** La Fase 2.5 resuelve esto compilando la misma versión (4.1.6) en todos los nodos.

**Verificación de Éxito:**
```bash
# Ejecutar en TODOS los nodos
mpirun --version | grep "Open MPI"

# Debe mostrar exactamente:
# mpirun (Open MPI) 4.1.6
```

### Alternativa: Usar Conda (Si la compilación falla)

Si tienes problemas compilando, puedes usar Conda para gestión de paquetes:

```bash
# Instalar Miniconda en TODOS los nodos
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
echo 'export PATH="$HOME/miniconda3/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Instalar OpenMPI específico
conda install -c conda-forge openmpi=4.1.6

# Verificar
mpirun --version
```

### Problema: "Permission denied" en SSH
```bash
# Verificar permisos
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Verificar configuración SSH
sudo nano /etc/ssh/sshd_config
# Asegurar que esté habilitado:
# PubkeyAuthentication yes
# AuthorizedKeysFile .ssh/authorized_keys
```

### Problema: NFS no monta
```bash
# Verificar servicios NFS
sudo systemctl status nfs-server  # En maestro
sudo systemctl status rpcbind     # En esclavos

# Verificar exports
showmount -e master-node
```

### Problema: MPI no encuentra hosts
```bash
# Verificar hostfile
cat /home/cluster/shared/hostfile

# Probar conectividad individual
for node in $(cat /home/cluster/shared/hostfile | awk '{print $1}'); do
    ssh cluster@$node hostname
done
```

---

## 📊 Métricas de Rendimiento Esperadas

Con esta configuración deberías obtener:

- **Speedup lineal** para problemas CPU-intensivos como números perfectos
- **Latencia SSH** < 1ms en red local
- **Throughput NFS** > 100 MB/s en red Gigabit
- **Eficiencia MPI** > 85% para trabajos distribuidos

---

## 🎯 Próximos Pasos

1. **Escalabilidad**: Agregar más nodos esclavos siguiendo la misma configuración
2. **Monitoreo**: Implementar Ganglia o Nagios para monitoreo continuo
3. **Queue System**: Implementar SLURM para gestión de trabajos
4. **Optimización**: Ajustar parámetros MPI para tu hardware específico

¡Tu cluster Beowulf MPI está listo para ejecutar aplicaciones paralelas!

---

## 🚀 Comandos Rápidos de Referencia

### Verificar Estado del Cluster
```bash
# Verificar versiones MPI
/home/cluster/shared/verificar_versiones.sh

# Monitorear recursos
/home/cluster/shared/monitor_cluster.sh

# Probar conectividad
mpirun -hostfile /home/cluster/shared/hostfile -np 4 hostname
```

### Ejecutar Programa de Números Perfectos
```bash
cd /home/cluster/shared

# Compilar
mpicc -o numeros_perfectos numeros_perfectos.c -lm -O3

# Ejecutar con 4 procesos
mpirun -hostfile hostfile -np 4 ./numeros_perfectos 1 100000

# Ejecutar con todos los núcleos disponibles
mpirun -hostfile hostfile --oversubscribe -np 8 ./numeros_perfectos 1 1000000
```

### Solución de Problemas
```bash
# Si hay problemas de versiones MPI
/home/cluster/shared/reparar_mpi.sh

# Reinstalar SSH keys
ssh-copy-id cluster@slave-node1
ssh-copy-id cluster@slave-node2

# Remontar NFS
sudo umount /home/cluster/shared
sudo mount -t nfs master-node:/home/cluster/shared /home/cluster/shared
```
