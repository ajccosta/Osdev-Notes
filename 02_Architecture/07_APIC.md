# APIC 

## What is APIC

APIC stands for *Advanced Programmable Interrupt Controller*, and it's the device used to manage incoming interrupts to a processor core. It replaces of the old PIC8259 (that remains still available), but it offers more functionality especially when dealing with SMP. In fact one of the limitations of the PIC was that it was able to deal with only one cpu at time, and this is also the main reason why the APIC was firstly introduced. 

It's worth noting that Intel later developed a version of the APIC called the SAPIC for the Itanium platform. These are referred to collectively as the *xapic*, so if this term is used in documentation know that it just means the local APIC.

## Types of APIC

There are two types of APIC:

* _Local APIC_: it is present in every processor core, it is responsible for handling incoming interrupts for *that core*. It can also be used for sending an IPI (inter-processor interrupt) to other cores, as well as generating some interrupts itself. Interrupts generated by the local APIC are controlled by the LVT (local vector table), which is part of the local APIC registers. The most interesting of these is the timer LVT, which we will take a closer look at in the timers chapter.
* _IO/APIC_: An IO APIC acts a 'gateway' for devices in the system to send interrupts to local APICs. Most personal computers will only have a single IO APIC, but more complex systems (like servers or industrial equipment) may contain multiple IO APICs. Each IO APIC has a number of input pins, which a connected device triggers when it wants to send an interrupt. When this pin is triggered, the IO APIC will send an interrupt to one (or many) local APICs, depending on the *redirection entry* for that pin. We can program these redirection entries, and they're presented as an array of memory-mapped registers. We'll look more at this later. In summary, an IO APIC allows us to route device interrupts to processor cores however we want.

Both types of APIC are accessed by memory mapped registers, with 32-bit wide registers. They both have well-known base addresses, but rather than hardcoding these they should be fetched from the proper places as firmware (or even the bootloader) may move these around before our kernel boots.

## Local APIC

When a system boots up, the cpu starts in PIC8259A emulation mode for legacy reasons. This simply means that instead of having the LAPIC/IO-APIC up and running, we have them working to emulate the old interrupt controller, so before we can use them properly we should to disable the PIC8259 emulation.

### Disabling The PIC8259

This part should be pretty straightforward, and we will not go deep into explaining the meaning of all command sent to it. The sequence of commands is: 

```c
void disable_pic() {
    outportb(PIC_COMMAND_MASTER, ICW_1);
    outportb(PIC_COMMAND_SLAVE, ICW_1);
    outportb(PIC_DATA_MASTER, ICW_2_M);
    outportb(PIC_DATA_SLAVE, ICW_2_S);
    outportb(PIC_DATA_MASTER, ICW_3_M);
    outportb(PIC_DATA_SLAVE, ICW_3_S);
    outportb(PIC_DATA_MASTER, ICW_4);
    outportb(PIC_DATA_SLAVE, ICW_4);
    outportb(PIC_DATA_MASTER, 0xFF);
    outportb(PIC_DATA_SLAVE, 0xFF);
}
```

The old x86 architecture had two PIC processor, and they were called "master" and "slave", and each of them has it's own data port and command port:

* Master PIC command port: `0x20` and data port: `0x21`.
* Slave PIC command port: `0xA0` and data port `0xA1`.

The ICW values are initialization commands (ICW stands for Initialization Command Words), every command word is one byte, and their meaning is: 

* ICW_1 (value `0x11`) is a word that indicates a start of initialization sequence, it is the same for both the master and slave pic. 
* ICW_2 (value `0x20` for master, and `0x28` for slave) are just the interrupt vector address value (IDT entries), since the first 31 interrupts are used by the exceptions/reserved, we need to use entries above this value (remember that each pic has 8 different irqs that can handle.
* ICW_3 (value `0x2` for master, `0x4` for slave) Is is used to indicate if the pin has a slave or not (since the slave pic will be connected to one of the interrupt pins of the master we need to indicate which one is), or in case of a slave device the value will be it's id. On x86 architectures the master irq pin connected to the slave is the second, this is why the value of ICW_M is 2
* ICW_4 contains some configuration bits for the mode of operation, in our case we just tell that we are going to use the 8086 mode. 
* Finally `0xFF` is used to mask all interrupts for the pic.

### Discovering the Local APIC

The first step needed to configure the LAPIC is getting access to it. The APIC registers are memory mapped, and to get their location we need to read the MSR (*model specific register*) that contains its base address, using the __rdmsr__ instruction. This instruction reads the content of the MSR specified in `ecx`, the result is placed in `eax` and `edx` (with `eax` containing the lower 32-bits, and `edx` container the upper 32-bits).

In our case the MSR that we need to read is called IA32_APIC_BASE and its value is `0x1B`.

This register contains the following information: 

* Bits 0:7: reserved.
* Bit 8: if set, it means that the processor is the Bootstrap Processor (BSP).
* Bits 9:10: reserved.
* Bit 11: APIC global enable. This bit can be cleared to disable the local APIC for this processor. Realistically there is no reason to do this on modern processors.
* Bits 12:31: Contains the base address of the local APIC for this processor core.
* Bits 32:63: reserved.

Note that the registers are given as a *physical address*, so to access these we will need to map them somewhere in the virtual address space. This is true for the addresses of any IO APICs we obtain as well. When the system boots, the base address is usually `0xFEE0000` and often this is the value we read from `rdmsr`. 

A complete list of local APIC registers is available in the Intel/AMD software development manuals, but the important ones for now are:

- Spurious Interrupt Vector: offset `0xF0`.
- EOI (end of interrupt): offset `0xB0`.
- Timer LVT: offset `0x320`.
- Local APIC ID: offset `0x20`.

### Enabling the Local APIC, and The Spurious Vector

The spurious vector register is also contains some miscellaneous config for the local APIC, including the enable/disable flag. This register has the following format:

| Bits  | Value                        |
|-------|------------------------------|
| 0-7   | Spurious vector              |
|  8    | APIC Software enable/disable | 
|  9    | Focus Processor checking     |
| 10-31 | Reserved                     |

The functions of the fields in the registers are as follows:

* Bits 0-7: They determine the vector number (IDT entry) for the spurious interrupt generated by the APIC.
* Bit 8: This bit acts a software toggle for enabling the local APIC, if set the local APIC is enabled.
* Bit 9: This is an optional feature not available on processors, but if set it indicates that some interrupts can be routed according to a list of priorities. This is an advanced topic and this bit can be safely left clear and ignored.

The Spurious Vector register is writable only in the first 9 bits, the rest is read only. In order to enable the lapic we need to set bit 8, and set-up a spurious vector entry for the idt. In modern processors the spurious vector can be any vector, however old CPUs have the upper 4 bits of the spurious vector forced to 1, meaning that the vector must be between `0xF0` and `0xFF`. For compatibility it's best to place the spurious vector in that range. Of course we need to set-up the corresponding idt entry with a function to handle it, but for now printing an error message is enough.

### Reading APIC Id and Version

The ID register contains the *physical id* of the local APIC in the system. This is unique and assigned by the firmware when the system is first started. Often this ID is used to distinguish each processor from the others due to them being unique. This register is allowed to be read/write in some processors, but it's recommended to treat it as read-only.

The version register contains some useful (if not really needed) information. Exploring this register is left as an exercise to the reader.

### Local Vector Table

The local vector table allows the software to specify how the local interrupts are delivered. 
There are 6 items in the LVT starting from offset `0x320` to `0x370`:

* *Timer*: used for controlling the local APIC timer. Offset: `0x320`.
* *Thermal Monitor*: used for configuring interrupts when certain thermal conditions are met. Offset: `0x330`.
* *Performance Counter*: allows an interrupt to be generated when a performance counter overflows. Offset: `0x340`.
* *LINT0*: Specifies the interrupt delivery when an interrupt is signaled on LINT0 pin. Offset: `0x350`.
* *LINT1*: Specifies the interrupt delivery when an interrupt is signaled on LINT1 pin. Offset: `0x360`.
* *Error*: configures how the local APIC should report an internal error, Offset `0x370`.

The `LINT0` and `LINT1` pins are mostly used for emulating the legacy PIC, but they may also be used as NMI sources. These are best left untouched until we have parsed the MADT, which will tell how the LVT for these pins should be programmed.

Most LVT entries use the following format, with the timer LVT being the notable exception. It's format is explained in the timers chapter. The thermal sensor and performance entries ignore bits 15:13.

| Bit      |  Description                                                                                 |
|----------|----------------------------------------------------------------------------------------------|
| 0:7      |  Interrupt Vector. This is the IDT entry we want to trigger when for this interrupt.       |
| 8:10     |  Delivery mode (see below)                                                                               |
| 11       |  Destination Mode, can be either physical or logical.                                        |
| 12       |  Delivery Status **(Read Only)**, whether the interrupt has been served or not.|
| 13       |  Pin polarity: 0 is active-high, 1 is active-low. |
| 14       |  Remote IRR **(Read Only)** used by the APIC for managing level-triggered interrupts. |
| 15       |  Trigger mode: 0 is edge-triggered, 1 is level-triggered. |
| 16       |  Interrupt mask, if it is 1 the interrupt is disabled, if 0 is enabled. |

The delivery mode field determines how the the APIC should present the interrupt to the processor. The fixed mode (0b000) is fine in almost all cases, the other modes are for specific functions or advanced usage.

### X2 APIC

The X2APIC is an extension of the XAPIC (the local APIC in it's regular mode). The main difference is the registers are now accessed via MSRs and some the ID register is expanded to use a 32-bit value (previously 8-bits). While we're going to look at how to use this mode, it's perfectly fine to not support it.

Checking whether the current processor supports the X2APIC or not can be done via `cpuid`. It will be under leaf 1, bit 21 in `ecx`. If this bit is set, the processor supports the X2APIC.

Enabling the X2APIC is done by setting bit 10 in the IA32_APIC_BASE MSR. It's important to note that once this bit is set, we cannot clear it to transition back to the regular APIC operation without resetting the system.

Once enabled, the local APIC registers are no longer memory mapped (trying to access them there is now an error) and can instead be accessed as a range of MSRs starting at `0x800`. Since each MSR is 64-bits wide, the offset used to access an APIC register is shifted right by 4 bits.

As an example, the spurious interrupt register is offset `0xF0`. To access the MSR version of this register we would shift it right by 4 (`0xF0 >> 4` = 0xF) and then add the base offset (`0x800`) to get the MSR we want. That means the spurious interrupt register is MSR `0x80F`.

Since MSRs are 64-bits, the upper 32 bits are zero on reads and ignored on writes. As always there is an exception to this, which is the ICR register (used for sending IPIs to other cores) which is now a single 64-bit register.

### Handling Interrupts

Once an interrupt for the local APIC is served, it won't send any further interrupts until the end of interrupt signal is sent. To do this write a 0 to the EOI register, and the local APIC will resume sending interrupts to the processor. This is a separate mechanism to the interrupt flag (IF), which also disables interrupts being served to the processor. It is possible to send EOI to the local APIC while IF is cleared (disabling interrupts) and no further interrupts will be served until IF is set again.

There are few exceptions where sending an EOI is not needed, this is mainly spurious interrupts and NMIs. 

The EOI can be sent at any time when handling an interrupt, but it's important to do it before returning with `iret`. If we enable interrupts and only receive a single interrupt, forgetting to send EOI may be the reason.

### Sending An IPI

If we want to support SMP (multiple cores) in our kernel, we will need a way to inform other cores that an event has occurred. This is typically done by sending an IPI. Note that IPIs don't carry any information about what event occurred, they simply indicate that *something* has happened. To send data about what the event is a struct is usually placed in memory somewhere, sometimes called a *mailbox*.

To send an IPI we need to know the local APIC ID of the core we wish to interrupt. We will also need a vector in the IDT set up for handling IPIs. With these two things we can use the ICR (interrupt command register).

The ICR is 64-bits wide and therefore we access it as two registers (a higher and lower half). The IPI is sent when the lower register is written to, so we should set up the destination in the higher half first, before writing the vector in the lower half.

This register contains a few fields but most can be safely ignored and left to zero. We're interested in bits 63:56 which is the ID of the target local APIC (in X2APIC mode it's bits 63:32) and bits 7:0 which contain the interrupt vector that will be served on the target core.

An example function might look like the following:

```c
void lapic_send_ipi(uint32_t dest_id, uint8_t vector) {
    lapic_write_reg(ICR_HIGH, dest_id << 24);
    lapic_write_reg(ICR_LOW, vector);
}
```

At this point the target core would receive an interrupt with the vector we specified (assuming that core is setup correctly).


There is also a shorthand field in the ICR which overrides the destination id. It's available in bits 19:18 and has the following definition:

- 0b00: no shorthand, use the destination id.
- 0b01: send this IPI to ourselves, no one else.
- 0b10: send this IPI to all lapics, including ourselves.
- 0b11: send this IPI to all lapics, but not ourselves.

## IOAPIC

The IOAPIC primary function is to receive external interrupt events from the systems, and is associated with I/O devices, and relay them to the local apic as interrupt messages, with the exception of the lapic timer, all external devices are going to use the IRQs provided by it (like it was done in the past by the PIC).

### Configure the IO-APIC

To configure the IO-APIC we need to: 

1. Get the IO-APIC base address from the MADT
2. Read the IO-APIC Interrupt Source Override table
3. Initialize the IO Redirection table entries for the interrupt we want to enable

### Getting IO-APIC address

Read IO-APIC information from MADT table (the MADT table is available within the RSDT data, we need to search for the MADT Table item type 1). The content of the MADT Table for the IO_APIC type is: 

| Offset | Length | Description                  |
|--------|--------|------------------------------|
| 2      | 1      | I/O Apic ID's                |
| 3      | 1      | Reserved (should be 0)       |
| 4      | 4      | I/O Apic Address             |
| 8      | 4      | Global System Interrupt Base |

The IO APIC ID field is mostly fluff, as we'll be accessing the io apic by it's mmio address, not it's ID.

The Global System Interrupt Base is the first interrupt number that the I/O Apic handles. In the case of most systems, with only a single IO APIC, this will be 0. 

To check the number of inputs an IO APIC supports:

```c
uint32_t ioapicver = read_io_apic_register(IOAPICVER);
size_t number_of_inputs = ((ioapicver >> 16) & 0xFF) + 1;
```

The number of inputs is encoded as bits 23:16 of the IOAPICVER register, minus one. 


### IO-APIC Registers

The IO-APIC has 2 memory mapped registers for accessing the other IO-APIC registers: 

| Memory Address | Mnemonic Name | Register Name      | Description                                  |
|----------------|---------------|--------------------|----------------------------------------------|
|   FEC0 0000h   | IOREGSEL      | I/O Register Select| Is used to select the I/O Register to access |
|   FEC0 0010h   | IOWIN         | I/O Window (data)  | Used to access data selected by IOREGSEL     |

And then there are 4 I/O Registers that can be accessed using the two above: 

| Name      | Offset   | Description                                            | Attribute | 
|:------------:|----------|--------------------------------------------------------|-----------|
| IOAPICID  | 00h      | Identification register for the IOAPIC                 |  R/W      |
| IOAPICVER | 01h      | IO APIC Version                                        |  RO       |
| IOAPICARB | 02h      | It contains the BUS arbitration priority for the IOAPIC|  RO       |
| IOREDTBL  | 03h-3fh  | The redirection tables (see the IOREDTBL paragraph)    |  RW       |


### Reading data from IO-APIC

There are basically two addresses that we need to use in order to write/read data from apic registers and they are: 

* IO_APIC_BASE address, that is the base address of the IOAPIC, called *register select* (or IOREGSEL)  and used to select the offset of the register we want to read
* IO_APIC_BASE + 0x10, called *i/o window register* (or IOWIN), is the memory location mapped to the register we intend to read/write specified by the contents of the *Register Select*

The format of the IOREGSEL is: 

| Bit     | Description                                                                                                          |
|---------|----------------------------------------------------------------------------------------------------------------------|
| 31:8    | Reserved                                                                                                             |
| 7:0     | APIC Register Address, they specifies the IOAPIC Registers to be read or written via the IOWIN Register              |

So basically if we want to read/write a register of the IOAPIC we need to: 

1. write the register index in the IOREGSEL register
2. read/write the content of the register selected in IOWIN register

The actual read or write operation is performed when IOWIN is accessed.
Accessing IOREGSEL has no side effects.

### Interrupt source overrides
They contain differences between the IA-PC standard and the dual 8250 interrupt definitions. The isa interrupts should be identity mapped into the first IO-APIC sources, but most of the time there will be at least one exception. This table contains those exceptions. 

An example is the PIT Timer is connected to ISA IRQ 0, but when apic is enabled it is connected to the IO-APIC interrupt input pin 2, so in this case we need an interrupt source override where the Source entry (bus source) is 0 and the global system interrupt is 2
The values stored in the IO Apic Interrupt source overrides in the MADT are:

| Offset | Length | Description                  |
|--------|--------|------------------------------|
| 2      | 1      | bus source (it should be 0)  |
| 3      | 1      | irq source                   |
| 4      | 4      | Global System Interrupt      |
| 8      | 2      | Flags                        |

* Bus source usually is constant and is 0 (is the ISA irq source), starting from ACPI v2 it is also a reserved field. 
* Irq source is the source IRQ pin
* Global system interrupt is the target IRQ on the APIC

Flags are defined as follows: 

* Polarity (*Lenght*: **2 bits**, *Offset*: *0*  of the APIC/IO input signals, possible values are:
    * 00 Use the default settings is active-low for level-triggered interrupts)
    * 01 Active High
    * 10 Reserved
    * 11 Active Low
* Trigger Mode (*Length*: **2 bits**, *Offset*: *2*) Trigger mode of the APIC I/O Input signals:
    * 00 Use the default settiungs (in the ISA is edge-triggered)
    * 01 Edge-triggered
    * 10 Reserved
    * 11 Level-Triggered
* Reserved (*Length*: **12 bits**, *Offset*: **4**) this must be 0


### IO Redirection Table (IOREDTBL)

They can be accessed vie memory mapped registers. Each entry is composed of 2 registers (starting from offset 10h). So for example the first entry will be composed by registers 10h and 11h.

The content of each entry is:

* The lower double word is basically an LVT entry, for their definition check the LVT entry definition
* The upper double word contains:
    - Bits 17 to 55: are Reserved
    - Bits 56 to 63: are the Destitnation Field, In physical addressing mode (see the destination bit of the entry) it is the local apic id to forward the interrupts to, for more information read the IO-APIC datasheet.

The number of items is stored in the IO-APIC MADT entry, but usually on modern architectures is 24.