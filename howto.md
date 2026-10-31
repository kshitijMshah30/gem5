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
    - I.src1.DepTrBMID, I.src2.DepTrBMID

    - FreeList          : list of free physical register that can be used
    - MT[I.src1].valid  : If the physical register that the logical reg maps to is ready [i.e we can read the reg file]
    - MT[I.src1].wait   : If the physical register that the logical reg maps to is "depnedent" on some long lat instr
    - MT[I.src1].PNR     : Physical Register Number of the logical register
    - Read valid, wait, and PNR for src1 & src2

    - MT[I.src1].DepTrBMID : Dependence Tracker Bit Mask ID

    - MT[I.rd].PNR = GetOne(FreeList)
    - MT[I.rd].valid = 0, MT[I.rd].PNR = 0

    [::IMP::]
    - IF the src of the instr are dependent .. tihs instr also must use the same ID
    - MT[I.rd].DepTrBMID = MT[I.src1].wait ? MT[I.src1].DepTrBMID : (MT[I.src2].wait ? MT[I.src2].DepTrBMID : 0)

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
            slot.src1.wait = Tag.type == Wait
            slot.src1.DepTRBMID = Tag.DepTRBMID  // ID allocated "OG" missing instr
        if (slot.src2.PNR == Tag.PNR)
            slot.src2.valid = Tag.type == WakeUp
            slot.src2.wait = Tag.type == Wait
            slot.src2.DepTRBMID = Tag.DepTRBMID // ID allocated "OG" missing instr
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
        - Tag broadcat 
            - Type : Wait
            - Rd
            - IQID
            - I.src1.wait ? I.src1.DepTrBMID : I.src2.DepTrBMID 
                - If we are waiting for dep instr ... dep instr on this instr should also update the same ID
                - So, if wait is due to src1, use its else src2 
                - What if Both are wait ? it fine ... if instr is put back in IQ when src1 is done, it will go back to WIB with src2.DepTrBMID
            - DepMat [I.src1.wait ? I.src1.DepTrBMID : I.src2.DepTrBMID] [I.WIBID] = 1
                - Mark itself dependent the "OG" instr
    - Else
        - Send to FU

Execution Logic:
    - Not Memory Access ? 
        - Execute Normally    - Each 
    - Memory Access ... ask cache 
        - Cache miss!

        - Each missed loads get a "DepTrBMID" : Dependences Tracker Bit Mask ID
        - DepMat : "N" vectors of len W
        - DepMat[I.DepTrBMID] : BitMask showing which instr depened on the instr with that DepTrBMID

        - Allocated a "LdDepTrkBM" 
            - [What if] No More LdDepTrkBM
                - Don't send Wait Tag Broadcast
                - Since we can't track the younger dep instr

        - Send Wait Tag Broadcast 
            - Wait 
            - I
            - LdDepTrBM
        
Write back: 
    - i.e instr is done 
    - Send Wakeup tag
    - MT[I.rd].valid = 1
    - MT[I.rd].wait  = 0
    
    - If instr was Memory access:
        - MT[I.rd].wait == 1 
            - i.e this memory access was a miss before, and the "wait" tag was broadcasted for it
            - i.e WIB needs to be told move these instr back

            - Send the BitMask to WIB
            - WIBSendBackBM = DepMat[MT[I.rd].DepTrBWID]
            - WIBSendBackEn = MT[I.rd].wait  // if wait was 1 ... a "Wait" tag for this was broadcasted before
            
        - if it was a miss before ... will need to tell the WIB 
            - How to find out ? 
                - when the miss happend ... the MT[I.rd].wait would be set to 1 ! 
                - We can use this as a check! 
            
        - If it wasn't a miss ... don't need to tell anyone  


    
        
        