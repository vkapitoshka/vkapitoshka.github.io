---
layout: post
title: Соответствие дисков VMware и дисков в ОС Windows
description: "Использование различных тегов в поста но через jekyll"
modified: 2020-03-04
author: Vlad Kap
tags: [VMware]
---
> На момент написание данной статьи используется VMware 6.5

> По роду своей деятельности мне достаточно часто приходится выполнять работы по увеличение дисков на виртуальных машинах VMware. В своей среде, мы в основном используем Windows ОС. В целом, эта процедура не сложная:
* Находим виртуальную машину в консоли VMware
* Выбираем диск, который нужно увеличить в консоли VMware
* Увеличиваем диск

> На этом работа можно сказать закончена и остается только увеличить диск в ОС виртуальной машины. Данную процедуру я описывать здесь не буду, так как она проста и не вызывает сложностей.

> Теперь рассмотрим такой вариан, у виртуальной машины 1 **SCSI** контроллер и два диска одинакового размера. Обычно ОС Windows диски нумеруются последовательно, т.е. диск 1 виртуальной машины будет соответствовать диск 0 в ОС Windows и так далее, но к сожалению это не всегда так, и лучше, убедиться что мы добавляем место именно тому диску на уровне Виртуализации, которому нужно.

На уровне виртуализации при подключении диска к одному и тому же **SCSI** контроллеру каждому диск выдается порядковый номер. Таким образом каждый диск имеет свой идентификатор он состоит из
номера SCSI контроллера,нумирация идет с 0, и номера под которым подключен диск. По умолчанию <u>первый диск</u> имеет адрес **0:0**, <u>второй</u> **0:1** и т.д.

![vm_Disk]({{ sute.url }}/assets/images/vm_Disk.jpg)
{: .image-right}
Максимальнок количество дисков для одного SCSI контроллера равно 16.

**Но как нам определить какой диск какой в ОС Windows?**

В этом нам поможет встроенная утилита **DISKPART**:

1. Открываем коносль CMD с повышением привилегий.
2. Выполняем команду **DISKPART** - в консоли вы перейдете в командную оболочку diskpart
3. Выполняем команду *list disk*, которая отобразит нам все диски подключенные к виртуальной машине

    ![List_Disks]({{ sute.url }}/assets/images/list_d.jpg)
    {: .image-right}

4. Выбираем один из дисков, для которого нам требуется определить его номер. Это делается командой *selest disk (0,1,2,3 и т.д.)*

    ![Select_Disks]({{ sute.url }}/assets/images/select_d.jpg)
    {: .image-right}

5. После того как нужный диск выбран, выполняем команду *detail disk*, чтобы посмотреть детали диска
6. В эти деталях нужно обратить внимание для пункт **Target** - это и есть вторая цифра в адресе диска в формате **0:0**, **0:1**...**0:15**

    ![Detail_Disks]({{ sute.url }}/assets/images/detail_d.jpg)
    {: .image-center}

7. Остается только перепроверить соответствие номеров в консоли **VMware** и в консоли **DISKPART** и выполнить стандартную процедуру увеличения диска описанную выше

**Но что делать, если диски с одинаковым размером были добавлены к Виртуальной машине и используют разные SCSI контроллеры?**

> Вроде бы все просто и понятно, но если будет добавлен диск на еще один SCSI контроллер например **1:0**. Тогда команда **DISKPART** для диска **0:0** и диска **1:0**, а если честно, то и для дисков **2:0** и **3:0** значение **Target** будет иметь значение 0 (к счастью, к виртуальной машине можно добавить только 4 **SCSI** контроллера **0,1,2,3**).

|SCSI 2 | SCSI 3 |
|:------:|:------:|
| ![Detail_Disks]({{ sute.url }}/assets/images/detail_d1.jpg) | ![Detail_Disks]({{ sute.url }}/assets/images/detail_d2.jpg)

> На картинке видно, что у данных дисков различный путь *Local Path*, но найти данный параметр в свойствах сетевого адаптера мне не удалось.

Как не странно, но оказалось, что VMWare монтирует виртуальные **SCSI** адаптеры всегда под одним и тем же номером порта для кажной виртуальной машины и зная это достаточно легко определить SCSI контроллер. Для 0 и 1 SCSI контроллера монтирование происходит под **PCI Slot 160** и **PCI Slot 256** соответсвенно.
Кстати эти же порты можно увидеть через PowerCLI командой: 
```yaml
get-vm test | Get-ScsiController | select Parent, @{N='SlotNumber';E={$_.ExtensionData.SlotInfo.PciSlotNumber}}
```

А вот для **SCSI** адаптеров 2 и 3 значения отличаются, но одинаковы для всех виртуальных машин, соответсвенно **PCI Slot 161** и **PCI Slot 193**

| SCSI 0 | SCSI 1 | SCSI 2 | SCSI 3 |
|:------:|:------:|:------:|:------:|
| ![Detail_Disks]({{ sute.url }}/assets/images/SCSI_0.jpg) | ![Detail_Disks]({{ sute.url }}/assets/images/SCSI_1.jpg) |![Detail_Disks]({{ sute.url }}/assets/images/SCSI_2.jpg) | ![Detail_Disks]({{ sute.url }}/assets/images/SCSI_3.jpg)

Для удобства составил таблицу:

| **SCSI адаптер** | **Номер порта в свойствах** | **Номер порта в PowerCLI** | **Путь в DiskPart** |
|:----------------:|:---------------------------:|:--------------------------:|:-------------------:|
| **SCSI 0** | PCI Slot 160 | 160 | PCIROOT(0)#PCI(1500)#PCI(0000)#SAS(P00T01L00) |
|----
| **SCSI 1** | PCI Slot 256 | 256  | PCIROOT(0)#PCI(1800)#PCI(0000)#SAS(P00T00L00) |
|----
| **SCSI 2** | PCI Slot 161 | 1184 | PCIROOT(0)#PCI(1501)#PCI(0000)#SAS(P00T00L00) |
|----
| **SCSI 3** | PCI Slot 193 | 1216 | PCIROOT(0)#PCI(1601)#PCI(0000)#SAS(P00T00L00) |
|----
{: rules="groups"}


Надеюсь данная статья поможет вам правильно выбрать какой диск следует увеличивать и убереж Вас от необходимости уменьшать  диск, о чем я напишу в следующей статье.