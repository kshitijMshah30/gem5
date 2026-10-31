WIB : Waiting Instruction Buffer

Goal : Implemenet the WIB on Gem5 o3 pipe

Core idea:
    - There exists long latency event [cache miss]
    - Naively adding younger instruction that depnedent on these long latency instr fills up the IQ
        - Size (IQ) << W [W is the instruction window size]
        - Size (ROB) = W ; Size (WIB) = W
    - Move the dependent instr to the WIB
    - Allows indepenet instr to be issued
    - Fwd progress

Dispatch:
    - Get's an IQ slot to go into
    - Get's a ROB ID [index in the ROB]
    - Get's a WIB ID [index in the WIB]

Re-name:
    - I.src1, I.src2, I.rd

    - FreeList          : list of free physical register that can be used
    - MT[I.src1].valid  : If the physical register that the logical reg maps to is ready [i.e we can read the reg file]
    - MT[I.src1].wait   : If the physical register that the logical reg maps to is "depnedent" on some long lat instr
    - MT[I.src].PNR     : Physical Register Number of the logical register
    - Read valid, wait, and PNR for src1 & src2

    - MT[I.rd].PNR = GetOne(FreeList)
    - MT[I.rd].valid = 0, MT[I.rd].PNR = 0

Insert:
    - But the follwoing info into the IQ:
        - {MT[I.src1].PNR, MT[I.src1].valid, MT[I.src1].wait}
        - {MT[I.src2].PNR, MT[I.src2].valid, MT[I.src2].wait}
        - MT[I.rd].PNR
        - I.op
    - Get a slot in the IQ, New_IQID
    - IQ[New_IQID].valid = 1

Wakeup:
    - Tag will contain the following feilds:
        - Type  : WakeUp, Wait
        - RdPNR : PNR of the destination reg
        - IQID  : ID of the IQ slot
    - IQ[.].valid : If the IQ entry is valid
    - IQ[.].issued : If the IQ entry has been issued.

    - Upon a tag broadcast, set the valid bits for the source that depends on the completing instruction
    - Normal tag [WakeUp] set mathcing src valid
    - Wait tag set matching wait valid

    - For slot in IQ
        if (slot.src1.PNR == Tag.PNR)
            slot.src1.valid = Tag.type == WakeUp
            slot.src.wait = Tag.type == Wait
        if (slot.src2.PNR == Tag.PNR)
            slot.src2.valid = Tag.type == WakeUp
            slot.src.wait = Tag.type == Wait
    - Free this instrs slot:
        - IQ[IQID].valid = Tag.type == WakeUp // Only valid on normal wakeup ... Wait tag ... cache miss!

    - Updates to Mapping Table [MT]:
        - MT[Tag.rd].valid = Tag.type == Wakeup
        - MT[Tag.rd].wait  = Tag.type == Wait
    
Select:
    - Bit instr to selected from IQ
        - S = {Phi}
        - for I in IQ
            if (I.src1.valid || I.src1.wait) && (I.src2.valid || I.src2.wait)
                S.append(I)
        - Sel instr from I
        - Issue it
        - Mark it as issued 
            IQ[I.IQID].issued = 1

Steering Logic:
    - If I.src1.wait || I.src2.wait
        - Send to WIB
        - Send {Wait, I.rd} tag
        - WIB[I.WIBID] 
    - Else
        - Send to FU

Execution Logic:
    - Not Memory Access ? 
        - Execute Normally
    - Memory Access ... ask cache 
        - Cache miss!
        - Allocated a "LdDepTrkBM" 
            - BitMask of size W
            - "1" => that ROB/WIBID instr depends on the load which has this vector
        - 


    
        
        