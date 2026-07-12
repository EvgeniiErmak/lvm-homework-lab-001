# Домашнее задание: Работа с LVM

Практическая работа по управлению логическими томами LVM, LVM snapshots, LVM cache, mirror, а также сравнение с btrfs и ZFS. Выполнено на Ubuntu 22.04.5 LTS.

Все манипуляции проводились **только на втором физическом диске (`/dev/nvme1n1`)**, диск с операционной системой (`/dev/nvme0n1`) не затрагивался.

## Скриншоты

Все скриншоты процесса выполнения находятся в папке [`screenshots/`](./screenshots).

## Структура выполненной работы

### 1. Подготовка диска
Полная очистка `/dev/nvme1n1` (wipefs, sgdisk zap) и разметка GPT на 6 разделов:
- `p1`, `p2` — под LVM (root/home/var)
- `p3`, `p4` — под btrfs + LVM cache
- `p5`, `p6` — под ZFS + L2ARC cache

### 2. LVM: root / home / var
- **`lab-vg`** создан на `/dev/nvme1n1p1` и `/dev/nvme1n1p2`
- **`lab-root`** — создан (20G), заполнен данными, уменьшен командами `resize2fs` + `lvreduce` до **8G**
- **`lab-home`** — том 30G, файловая система **xfs**
- **`lab-var`** — том 10G в режиме **mirror** (`lvcreate --type mirror -m1`), реплика распределена на два разных физических раздела (`nvme1n1p1` / `nvme1n1p2`)
- Точки монтирования прописаны в `/etc/fstab` с разными опциями (`noatime`, `data=writeback` и т.д.)

### 3. LVM Snapshots (`/mnt/lab-home`)
1. Сгенерированы тестовые файлы
2. Снят снапшот (`lvcreate -s`)
3. Часть файлов удалена
4. Выполнено восстановление через `lvconvert --merge`, файлы вернулись

### 4. btrfs + LVM cache
- `btrfs-vg` создан на `/dev/nvme1n1p3` (данные) и `/dev/nvme1n1p4` (кэш)
- Настроен `cache-pool` (`lvconvert --type cache-pool`) и подключен к тому данных (`lvconvert --type cache`)
- Создана файловая система **btrfs**, смонтирована в `/opt/btrfs`
- Продемонстрированы subvolume и snapshot (`btrfs subvolume snapshot`)

### 5. ZFS с кэшем и снапшотами
- Пул `labtank` создан на `/dev/nvme1n1p5`
- Добавлен L2ARC cache-vdev на `/dev/nvme1n1p6` (`zpool add labtank cache`)
- Датасет `labtank/opt`, смонтирован в `/opt/zfs`
- Продемонстрирован снапшот (`zfs snapshot`) и rollback (`zfs rollback`)

## Итоговая структура томов

| Точка монтирования | Устройство | Тип | ФС | Размер |
|---|---|---|---|---|
| `/mnt/lab-root` | `/dev/lab-vg/lab-root` | LVM LV | ext4 | 8G |
| `/mnt/lab-home` | `/dev/lab-vg/lab-home` | LVM LV | xfs | 30G |
| `/mnt/lab-var` | `/dev/lab-vg/lab-var` | LVM mirror | ext4 | 10G |
| `/opt/btrfs` | `/dev/btrfs-vg/btrfs-data` | LVM + cache | btrfs | 95G |
| `/opt/zfs` | `labtank/opt` | ZFS + L2ARC | zfs | 78G |
