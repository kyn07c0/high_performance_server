### 1. Настройка процессора и управления частотой
Глубокое отключение энергосберегающих функций  
В BIOS/UEFI отключаем:
```text
- Intel Turbo Boost Technology → Disabled
- Enhanced Intel SpeedStep Technology → Disabled
- CPU C-States → Disabled
- Package C-State Limit → C0/C1
- CPU Thermal Monitoring → Disabled (только для систем с хорошим охлаждением!)
- P-States → Disabled
```

В Linux дополнительно:  

Полное отключение всех состояний C-State кроме C0/C1
```bash
echo 1 | sudo tee /sys/devices/system/cpu/cpu*/cpuidle/state*/disable
```
Отключение P-States для максимальной стабильности частоты
```bash
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```
Запрет на автоматическое управление частотой
```bash
echo 0 | sudo tee /sys/devices/system/cpu/intel_pstate/status
```
Фиксируем частоту через cpupower (пример для Xeon Gold 6248)
```bash
sudo cpupower frequency-set -d 2.4GHz -u 2.4GHz --min 2.4GHz --max 2.4GHz
```
# Проверяем результат
cpupower frequency-info
Мониторинг thermal и power
bash
# Установка утилит мониторинга
sudo apt-get install lm-sensors powertop

# Настройка
sudo sensors-detect
sudo service kmod start

# Постоянный мониторинг
watch -n 1 'cat /proc/cpuinfo | grep "MHz" && sensors | grep Core'
2. Продвинутая привязка процессов и изоляция ядер
Детальная стратегия изоляции
Для системы с 2x Xeon Gold 6248 (20 ядер/40 потоков на сокет):

bash
# Топология системы
numactl -H

# Разделяем ядра:
# - Ядра 0-5: Системные процессы, IRQ (нода 0)
# - Ядра 6-9: Сетевые прерывания, softirq (нода 0)  
# - Ядра 10-19: Приложение "танк" (нода 0)
# - Ядра 20-25: Системные процессы (нода 1)
# - Ядра 26-29: Сетевые прерывания (нода 1)
# - Ядра 30-39: Резерв или второе приложение (нода 1)
Изоляция через ядро Linux
В /etc/default/grub:

bash
GRUB_CMDLINE_LINUX="... isolcpus=10-19,30-39 nohz_full=10-19,30-39 rcu_nocbs=10-19,30-39"
Обновляем и перезагружаем:

bash
sudo update-grub2
sudo reboot
Привязка системных процессов
bash
# Привязываем systemd и системные процессы к системным ядрам
sudo systemctl set-property user.slice AllowedCPUs=0-5,20-25
sudo systemctl set-property system.slice AllowedCPUs=0-5,20-25
sudo systemctl set-property init.scope AllowedCPUs=0-5,20-25

# Привязываем kthreads
for i in $(pgrep kthread); do
    taskset -cp 0-5,20-25 $i > /dev/null 2>&1
done
3. Детальная настройка прерываний (IRQ)
Полное управление IRQ балансировкой
bash
# Отключаем автоматическую балансировку
sudo systemctl stop irqbalance
sudo systemctl disable irqbalance

# Создаем скрипт для ручного распределения IRQ
sudo tee /usr/local/bin/irq_affinity.sh << 'EOF'
#!/bin/bash

# Маски для ядер (hex)
SYS_CORES_MASK0="3f"    # Ядра 0-5 (нода 0)     - 00111111
NET_CORES_MASK0="3c0"   # Ядра 6-9 (нода 0)     - 1111000000  
SYS_CORES_MASK1="3f00000" # Ядра 20-25 (нода 1) - 111111000000000000000000
NET_CORES_MASK1="3c0000000" # Ядра 26-29 (нода 1)

# Функция для сетевых интерфейсов
set_net_irq_affinity() {
    local interface=$1
    local mask=$2
    
    # Получаем IRQ номера для интерфейса
    irq_list=$(grep "$interface" /proc/interrupts | awk '{print $1}' | sed 's/://')
    
    for irq in $irq_list; do
        if [ -f "/proc/irq/$irq/smp_affinity" ]; then
            echo "Setting IRQ $irq to mask $mask for $interface"
            printf "%x" $mask | sudo tee /proc/irq/$irq/smp_affinity
        fi
    done
}

# Применяем настройки
set_net_irq_affinity "ens1f0" $NET_CORES_MASK0
set_net_irq_affinity "ens1f1" $NET_CORES_MASK0
set_net_irq_affinity "ens2f0" $NET_CORES_MASK1
set_net_irq_affinity "ens2f1" $NET_CORES_MASK1

# Прочие IRQ распределяем по системным ядрам
for irq in $(ls /proc/irq | grep -v default); do
    if [ -f "/proc/irq/$irq/smp_affinity" ] && \
       ! grep -q "ens" /proc/interrupts | grep -q "$irq:"; then
        echo "Setting system IRQ $irq to SYS mask"
        echo "$SYS_CORES_MASK0" | sudo tee /proc/irq/$irq/smp_affinity
    fi
done
EOF

sudo chmod +x /usr/local/bin/irq_affinity.sh
4. Продвинутый тюнинг sysctl параметров
Сетевой стек - оптимизация для 100GbE
Создаем /etc/sysctl.d/99-highperf.conf:

bash
# ===== БАЗОВЫЕ НАСТРОЙКИ СЕТИ =====
# Увеличиваем максимальное количество соединений
net.core.somaxconn = 65536
net.core.netdev_max_backlog = 300000
net.core.optmem_max = 134217728

# Размеры буферов приема/передачи
net.core.rmem_default = 134217728
net.core.wmem_default = 134217728
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728

# ===== TCP НАСТРОЙКИ =====
# TCP буферы
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_mem = 134217728 134217728 134217728

# TCP алгоритмы
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_no_metrics_save = 1
net.ipv4.tcp_moderate_rcvbuf = 1
net.ipv4.tcp_adv_win_scale = 1

# TCP тайм-ауты и повторные передачи
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_syn_backlog = 262144

# TCP congestion control (для высокоскоростных сетей)
net.ipv4.tcp_congestion_control = bbr

# ===== БЕЗОПАСНОСТЬ И ФИЛЬТРАЦИЯ =====
# Netfilter/Conntrack
net.netfilter.nf_conntrack_max = 2000000
net.netfilter.nf_conntrack_tcp_timeout_established = 600
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 10
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 10

# ===== ПАМЯТЬ И VIRTUAL MEMORY =====
vm.swappiness = 1
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.dirty_expire_centisecs = 1000
vm.dirty_writeback_centisecs = 100
vm.vfs_cache_pressure = 1000
vm.zone_reclaim_mode = 0
vm.nr_hugepages = 1024

# ===== ФАЙЛОВАЯ СИСТЕМА =====
fs.file-max = 5000000
fs.aio-max-nr = 1048576
5. NUMA оптимизация - глубокое погружение
Анализ топологии NUMA
bash
#!/bin/bash
# numa_analysis.sh - детальный анализ NUMA топологии

echo "=== NUMA TOPOLOGY ==="
numactl -H

echo -e "\n=== NUMA MEMORY USAGE ==="
numastat

echo -e "\n=== CPU AFFINITY ==="
for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    cpu_num=$(basename $cpu | sed 's/cpu//')
    node=$(cat $cpu/node*/* 2>/dev/null | head -n1)
    echo "CPU $cpu_num -> NUMA Node $node"
done

echo -e "\n=== PCI DEVICE NUMA AFFINITY ==="
for device in /sys/bus/pci/devices/*; do
    if [ -f "$device/numa_node" ]; then
        device_name=$(basename $device)
        numa_node=$(cat $device/numa_node)
        echo "PCI $device_name -> NUMA Node $numa_node"
    fi
done
Оптимизация памяти для NUMA
bash
# Принудительное распределение памяти
echo 0 | sudo tee /proc/sys/vm/zone_reclaim_mode

# Запуск приложения с оптимальными NUMA настройками
numactl --cpunodebind=0 --membind=0 --interleave=0 \
        --physcpubind=10-19,30-39 \
        /path/to/application

# Pre-allocation огромных страниц для нужд приложения
echo 1024 | sudo tee /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 | sudo tee /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
6. Новые методики оптимизации
Оптимизация подсистемы ввода-вывода
Тюнинг I/O scheduler и queue depth:

bash
# Для NVMe дисков
echo 0 > /sys/block/nvme0n1/queue/nomerges
echo 1024 > /sys/block/nvme0n1/queue/nr_requests
echo "none" > /sys/block/nvme0n1/queue/scheduler

# Для SSD дисков
echo "kyber" > /sys/block/sda/queue/scheduler
echo 256 > /sys/block/sda/queue/nr_requests
echo 128 > /sys/block/sda/queue/read_ahead_kb

# Увеличиваем лимиты AIO
echo 1048576 > /proc/sys/fs/aio-max-nr
echo 65536 > /proc/sys/fs/aio-nr
Оптимизация сетевого стека для zero-copy
bash
# Включение zero-copy для сетевых операций
echo 1 > /proc/sys/net/core/busy_poll
echo 50 > /proc/sys/net/core/busy_read

# Настройка сетевых карт для максимальной производительности
ethtool -G ens1f0 rx 4096 tx 4096
ethtool -G ens1f1 rx 4096 tx 4096
ethtool -K ens1f0 tso on gso on gro on lro off
ethtool -K ens1f1 tso on gso on gro on lro off
ethtool -C ens1f0 rx-usecs 0 rx-frames 0
ethtool -C ens1f1 rx-usecs 0 rx-frames 0
Мониторинг и профилирование производительности
Установка и настройка продвинутых инструментов:

bash
# Установка необходимых пакетов
sudo apt-get install -y perf iperf3 sysstat nicstat numactl bpftrace

# Постоянный мониторинг производительности
sudo tee /usr/local/bin/perf_monitor.sh << 'EOF'
#!/bin/bash
while true; do
    clear
    echo "=== REAL-TIME PERFORMANCE MONITOR ==="
    echo "CPU: $(uptime)"
    echo "Memory: $(free -h | grep Mem)"
    echo ""
    echo "=== NETWORK ==="
    cat /proc/net/softnet_stat | head -n 5
    echo ""
    echo "=== IRQ BALANCE ==="
    grep -H . /proc/irq/*/smp_affinity | head -n 10
    echo ""
    echo "=== NUMA STATS ==="
    numastat -c | head -n 10
    sleep 2
done
EOF
Оптимизация памяти и huge pages
bash
# Динамическое управление huge pages
echo 'vm.nr_overcommit_hugepages = 512' >> /etc/sysctl.conf
echo 'vm.hugetlb_shm_group = 0' >> /etc/sysctl.conf

# Pre-allocation 1GB huge pages (если поддерживается)
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Настройка transparent huge pages (отключаем для детерминизма)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
Security vs Performance баланс
bash
# Отключаем security features, влияющие на производительность
echo 0 > /proc/sys/kernel/randomize_va_space
echo 0 > /proc/sys/kernel/kptr_restrict

# Настройка SELinux/AppArmor для минимального overhead
sudo setenforce 0
echo 'kernel.randomize_va_space = 0' >> /etc/sysctl.conf
Контейнерная оптимизация
Настройка Docker для высоких нагрузок:

json
{
  "data-root": "/opt/containers/docker",
  "storage-driver": "overlay2",
  "iptables": false,
  "ip-masq": false,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 1000000,
      "Soft": 1000000
    }
  },
  "cpu-rt-runtime": 950000,
  "cpu-rt-period": 1000000
}
Power management тюнинг
bash
# Принудительное включение максимальной производительности
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Отключение управления питанием PCIe
for device in /sys/bus/pci/devices/*; do
    echo performance | sudo tee $device/power/control 2>/dev/null
done

# Управление питанием дисков
for disk in /sys/block/sd*; do
    echo 0 | sudo tee $disk/queue/iosched/slice_idle 2>/dev/null
    echo 0 | sudo tee $disk/queue/rotational 2>/dev/null
done
7. Автоматизация и валидация
Скрипт автоматической проверки оптимизаций
bash
#!/bin/bash
# validate_optimizations.sh

validate_optimization() {
    local check=$1
    local expected=$2
    local actual=$3
    local description=$4
    
    if [ "$actual" = "$expected" ]; then
        echo "✓ $description"
    else
        echo "✗ $description (expected: $expected, got: $actual)"
    fi
}

echo "Validating system optimizations..."

# Проверка CPU governor
validate_optimization "CPU Governor" "performance" \
    $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor) \
    "CPU frequency governor"

# Проверка Turbo Boost
validate_optimization "Turbo Boost" "1" \
    $(cat /sys/devices/system/cpu/intel_pstate/no_turbo) \
    "Turbo Boost disabled"

# Проверка изолированных ядер
validate_optimization "Isolated CPUs" "10-19,30-39" \
    $(cat /proc/cmdline | grep -o 'isolcpus=[^ ]*' | cut -d= -f2) \
    "CPU isolation"

# Проверка huge pages
validate_optimization "Huge Pages" "never" \
    $(cat /sys/kernel/mm/transparent_hugepage/enabled | awk -F'[\\[\\]]' '{print $2}') \
    "Transparent Huge Pages"
