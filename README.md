# Oberon-allow-importing-any-number-of-modules
Note: In this repository, the term "Project Oberon 2013" refers to a re-implementation of the original "Project Oberon" on an FPGA development board around 2013, as published at www.projectoberon.com.

Project Oberon 2013 (see http://www.projectoberon.com) has a restriction that only 15 modules can be imported. Usually, this poses no problem, as it is good programming practice to define the module hierarchy such that only a small number of modules is (directly) imported by any given module.

However, this upper limit also includes modules from which types are referenced *indirectly*. Such modules do not necessarily appear in the import list of a module, i.e. their imports may be *hidden*. In deep module hierarchies, lifting this restricton may therefore become desirable.

The code in this repository increases the maximum number of modules that can be imported (directly or indirectly) to 63.

This is achieved by adjusting all instructions and data elements concerned to use 6 bits instead of 4 bits for the module number (mno) and by adapting their associated fixup mechanisms in the module loader accordingly.

**1. BL instructions for external procedure calls**

Change BL instructions for external procedure calls, as generated by the compiler (see *ORG.Call*) from

     | BL (4) | cond (4) | mno (4) | pno (8) | pc-fixorgP (12) |    (max mno = 15, max pno = 255, max disp = 2^12-1 = 4K-1 words)

to

     | BL (4) | mno (6) | pno (8) | pc-fixorgP (14) |    (max mno = 63, max pno = 255, max disp = 2^14-1 = 16K-1 words)

which the module loader will fix up to

     | BL (4) | cond (4) | offset relative to PC (24) |    (max offset = 2^24-1 = 16M-1 words)

Thus, the *cond* field is eliminated from the instruction, as generated by the compiler, and re-inserted by the loader.

This is possible because BL instructions only ever use *cond* = 7 (=always) and the module loader already inserts *BL 7* as a hardcoded constant in the fixed up instruction anyway (see the fixup code at the end of procedure *Modules.Load*).

The new (compile-time) instruction encoding also increases the maximum displacement between two BL instructions in the fixup chain from 2^12-1 = 4K-1 words to 2^14-1 = 16K-1 words. Given that the array *ORG.code* holds only 8K words, this eliminates the need for an extra check in *ORG.Call* whether a fixup is possible (*IF pc-fixorgP < 1000H*).

We keep the first 4 bits of the BL instruction so that *ORTool.DecObj* can recognize it as such. An alternative would have been to keep only the first 2 bits, leaving 8 bits for the module number (mno). But we opted to keep the U and V bits as well, as they are interpreted by *ORTool.DecObj*. Using 6 bits for the module number is no real limitation, as 2^6-1 = 63 imported modules should be enough for most purposes, even when taking into account indirectly imported modules.

**2. LD instructions for loading the static base of a module**

Change LD instructions for loading the static base of a module, as generated by the compiler (see *ORG.GetSB*) from

     | LD (4) | reg (4) | mno (4) | pc-fixorgD (20) |    (max mno = 15, max disp = 2^20-1 = 1M-1 words)

to

     | LD (4) | reg (4) | mno (6) | pc-fixorgD (18) |    (max mno = 63, max disp = 2^18-1 = 256K-1 words)

which the module loader will fix up to

     | LD (4) | reg (4) | MT (4) | offset for imported module in MT table (20) |    (max offset = 2^20-1 = 1M-1 words)

**3. Entries in type descriptor extension tables**

Change entries in type descriptor extension tables, as generated by the compiler (see *ORG.Q*) from

     | unused (4) | mno (4) | TDadr/exno (12) | pc-fixorgT (12) |    (max mno = 15, max TDadr/exno = 4095, max disp = 2^12-1 = 4K-1 words)

to

     | unused (2) | mno (6) | TDadr/exno (12) | pc-fixorgT (12) |    (max mno = 63, max TDadr/exno = 4095, max disp = 2^12-1 = 4K-1 words)

which the module loader will fix up to

     | absolute memory address of type descriptor (32) |

Extension table entries generated by the compiler already allowed 8 bits for the module number (mno), but the module loader previously extracted only the least significant 4 bits during the fixup phase. It now extracts 6 bits.

**4. Entries in method tables (Extended Oberon only)**

Change entries in method tables, as generated by the compiler (see *ORG.BuildTD*) from

     | unused (4) | mno (4) | mthadr/exno (14) | pc-fixorgM (10) |    (max mno = 15, max disp = 2^10-1 = 1023 words)

to

     | mno (6) | mthadr/exno (16) | pc-fixorgM (10) |    (max mno = 63, max disp = 2^10-1 = 1023 words)

which the module loader will fix up to

     | absolute memory address of method (32) |

Method table entries generated by the compiler already allowed 8 bits for the module number (mno), but the module loader previously extracted only the least significant 4 bits during the fixup phase. It now extracts 6 bits.

Note: We use 16 bits for the mthadr field (mthadr is the offset from mod.code in bytes), because 2^16 = 64KB. This is enough, since the code array of a module is only 8K words = 32K bytes. Alternatively, one could have decided to let the compiler (in ORG.BuildTD) insert the mthadr in words, use only 14 bits for the mthadr and adapt the loader/linker accordingly. But there was no need to do so.

**5. BL instructions for external super calls (Extended Oberon only)**

Change BL instructions for external super calls, as generated by the compiler (see *ORG.Call*) from

     | BL (4) | cond (4) | mno (4) | pno (8) | pc-fixorgP (12) |    (max mno = 15, max pno = 255, max disp = 2^12-1 = 4K-1 words)

to

     | BL (4) | mno (6) | pno (8) | pc-fixorgP (14) |    (max mno = 63, max pno = 255, max disp = 2^14-1 = 16K-1 words)

which the module loader will fix up to

     | BL (4) | cond (4) | offset relative to PC (24) |    (max offset = 2^24-1 = 16M-1 words)

# Alternatives considered

*Variant 1*

     | BL (4) | cond (4) | pno (8) | pc-fixorgP[mno] (16) |    (max pno = 255, fixorgP[mno] = heading of fixup chain for module with module number mno)

*Variant 2*

     | BL (4) | cond (4) | mno (5) | pno (7) | pc-fixorgP (12) |   (max mno = 31, max pno = 127, max disp = 4095 words)

*Variant 3*

     | BL (4) | cond (4) | mno (6) | pno (6) | pc-fixorgP (12) |   (max mno = 63, max pno = 63, max disp = 4095 words)

*Variant 4*

     | BL (4) | cond (4) | mno (7) | pno (5) | pc-fixorgP (12) |   (max mno = 127, max pno = 31, max disp = 4095 words)

*Variant 5*

     | BL (4) | cond (4) | mno (8) | pno (8) | pc-fixorgP (8) |   (max mno = 255, max pno = 255, max disp = 255 words)

In variant 1, the module number (mno) is no longer part of the BL instruction, as generated by the compiler. Instead, a *separate* fixup list is constructed for each imported module, the heading of which is stored along with the description of the import in the object file. This makes the module number (mno) implicit, i.e. the module loader "knows" the module number, since imported modules are processed in ascending order. This is similar to the fixup mechanism used in the A2 operating system (http://cas.inf.ethz.ch/projects/a2). However, in Project Oberon 2013, the *cond* field in BL instructions only ever uses a value of 7 (=always). Thus, one can simply eliminate the *cond* field at compile time (to make room for additional bits for the module number *mno*) and re-insert it at load time.

Variants 2 - 5 increase the number of bits used to encode the module number (mno) *at the expense* of *reducing* the number of bits used for the other fields. We have refrained from adopting any of these variants for several reasons:

* First, we have a solution that does not require reducing the number of bits of other fields in the instruction (apart from "abusing" the *cond* field to hold additional bits of the module number during compilation).

* Second, the procedure number (pno) is, in fact, the export number (exno) of an exported procedure. When exporting objects, the compiler consecutively numbers *all* exported entities, not just procedures. Thus, it is possible for the *pno* to be higher than the total number of exported procedures (because there may be other exported objects in the same module such as types or variables). Variants 2, 3 and 4 would limit the pno to 127, 63 and 31, respectively.

* Third, the module number (mno) not only counts the explicitly imported modules, but also modules from which types are referenced only *indirectly* via so-called re-export (and re-import) conditions. By using 8 bits for the module number (mno), we should be on the safe side for most practical purposes.

* Finally, the displacement between any two BL instructions in the code section of a module can, of course, be larger than 256 words = 1 KB. Variant 5 which would limit the displacement to less than 1 KB, which is unacceptable.

# Instructions for converting an existing Project Oberon 2013 system to be able to import more than 15 modules

**PREREQUISITES**: A current version of Project Oberon 2013 (see http://www.projectoberon.com). If you use Extended Oberon (see http://github.com/andreaspirklbauer/Oberon-extended), the functionality is already implemented.

------------------------------------------------------

**STEP 1**: Download and import the files of this repository to your Oberon system

Download all files from the [**Sources/FPGAOberon2013**](Sources/FPGAOberon2013) (for Project Oberon 2013) directory of this repository.

Convert the *source* files to Oberon format (Oberon uses CR as line endings) using the command [**dos2oberon**](dos2oberon), also available in this repository (example shown for Linux or MacOS):

     for x in *.Mod ; do ./dos2oberon $x $x ; done

For Project Oberon 2013, the new files are:

     ORB.Mod
     ORG.Mod
     ORL.Mod
     Modules.Mod

Import the files to your Oberon system. If you use an emulator, click on the *PCLink1.Run* link in the *System.Tool* viewer, copy the files to the emulator directory, and execute the following command on the command shell of your host system:

     cd oberon-risc-emu
     for x in *.Mod ; do ./pcreceive.sh $x ; sleep 0.5 ; done

------------------------------------------------------

**STEP 2:** Build a cross-development toolchain by compiling the "new" compiler and boot linker/loader on the "old" system

     ORP.Compile ORS.Mod/s ORB.Mod/s ~
     ORP.Compile ORG.Mod/s ORP.Mod/s ~
     ORP.Compile ORL.Mod/s ~
     System.Free ORTool ORP ORG ORB ORS ORL ~

------------------------------------------------------

**STEP 3:** Use the cross-development toolchain to build a new version of your Oberon system

     ORP.Compile Kernel.Mod/s FileDir.Mod/s Files.Mod/s Modules.Mod/s ~
     ORL.Link Modules ~
     ORL.Load Modules.bin ~

     ORP.Compile Input.Mod/s Display.Mod/s Viewers.Mod/s ~
     ORP.Compile Fonts.Mod/s Texts.Mod/s Oberon.Mod/s ~
     ORP.Compile MenuViewers.Mod/s TextFrames.Mod/s ~
     ORP.Compile System.Mod/s Edit.Mod/s Tools.Mod/s ~

Re-compile the Oberon compiler itself before (!) restarting the system:

    ORP.Compile ORS.Mod/s ORB.Mod/s ORG.Mod/s ORP.Mod/s ~
    ORP.Compile ORTool.Mod/s ORL.Mod/s ~

The last step is necessary because the modified version of your Oberon system now uses a different Oberon object file format (the file format itself is actually unchanged, but the BL and LD instructions contained in the object file have a different format). If you don't re-compile the compiler before restarting the Oberon system, you won't be able to start it afterwards (a TRAP is generated by the module loader during the fixup phase)!

------------------------------------------------------

**STEP 4:** Restart the Oberon system

You are now running the modified version of your Oberon system that allows importing more than 15 modules.

Re-compile any other modules that you may have on your system.

# Appendix: Changes made to Project Oberon 2013

**ORB.Mod**

```diff
--- FPGAOberon2013/ORB.Mod	2020-01-26 13:48:22.000000000 +0100
+++ Oberon-allow-importing-any-number-of-modules/Sources/FPGAOberon2013/ORB.Mod	2020-02-06 16:58:30.000000000 +0100
@@ -1,4 +1,4 @@
-MODULE ORB;   (*NW 25.6.2014  / 1.3.2019  in Oberon-07*)
+MODULE ORB;   (*NW 25.6.2014  / 1.3.2019  in Oberon-07 / AP 6.2.20*)
   IMPORT Files, ORS;
   (*Definition of data types Object and Type, which together form the data structure
     called "symbol table". Contains procedures for creation of Objects, and for search:
@@ -136,7 +136,8 @@
     IF obj = NIL THEN  (*insert new module*)
       NEW(mod); mod.class := Mod; mod.rdo := FALSE;
       mod.name := name; mod.orgname := orgname; mod.val := key;
-      mod.lev := nofmod; INC(nofmod); mod.type := noType; mod.dsc := NIL; mod.next := NIL;
+      IF nofmod < 64 THEN mod.lev := nofmod; INC(nofmod) ELSE ORS.Mark("too many imports") END ;
+      mod.type := noType; mod.dsc := NIL; mod.next := NIL;
       obj1.next := mod; obj := mod
     ELSE (*module already present*)
       IF non THEN ORS.Mark("invalid import order") END
```

**ORG.Mod**

```diff
--- FPGAOberon2013/ORG.Mod  2019-05-30 17:58:14.000000000 +0200
+++ Oberon-allow-importing-any-number-of-modules/Sources/FPGAOberon2013/ORG.Mod  2020-03-12 18:50:03.000000000 +0100
@@ -1,4 +1,4 @@
-MODULE ORG; (* N.Wirth, 16.4.2016 / 4.4.2017 / 31.5.2019  Oberon compiler; code generator for RISC*)
+MODULE ORG; (* N.Wirth, 16.4.2016 / 4.4.2017 / 31.5.2019  Oberon compiler; code generator for RISC / AP 12.3.20*)
   IMPORT SYSTEM, Files, ORS, ORB;
   (*Code generator for Oberon compiler for RISC processor.
      Procedural interface to Parser OSAP; result in array "code".
@@ -7,7 +7,7 @@
   CONST WordSize* = 4;
     StkOrg0 = -64; VarOrg0 = 0;  (*for RISC-0 only*)
     MT = 12; SP = 14; LNK = 15;   (*dedicated registers*)
-    maxCode = 8000; maxStrx = 2400; maxTD = 160; C24 = 1000000H;
+    maxCode = 8800; maxStrx = 3200; maxTD = 160; C24 = 1000000H;
     Reg = 10; RegI = 11; Cond = 12;  (*internal item modes*)
 
   (*frequently used opcodes*)  U = 2000H; V = 1000H;
@@ -76,11 +76,21 @@
     code[pc] := ((op * 10H + a) * 10H + b) * 100000H + (off MOD 100000H); INC(pc)
   END Put2;
 
+  PROCEDURE Put2a(op, a, mno, disp: LONGINT);
+  BEGIN (*emit load/store instruction to be fixed up by loader*)
+    code[pc] := ((op * 10H + a) * 40H + mno) * 40000H + (disp MOD 40000H); INC(pc)
+  END Put2a;
+
   PROCEDURE Put3(op, cond, off: LONGINT);
   BEGIN (*emit branch instruction*)
     code[pc] := ((op+12) * 10H + cond) * 1000000H + (off MOD 1000000H); INC(pc)
   END Put3;
 
+  PROCEDURE Put3a(op, mno, pno, disp: LONGINT);
+  BEGIN (*emit BL instruction to be fixed up by loader; 0 <= mno < 64*)
+    code[pc] := (((op+12) * 40H + mno) * 100H + pno) * 4000H + (disp MOD 4000H); INC(pc)
+  END Put3a;
+
   PROCEDURE incR;
   BEGIN
     IF RH < MT-1 THEN INC(RH) ELSE ORS.Mark("register stack overflow") END
@@ -147,7 +157,7 @@
   PROCEDURE GetSB(base: LONGINT);
   BEGIN
     IF version = 0 THEN Put1(Mov, RH, 0, VarOrg0)
-    ELSE Put2(Ldr, RH, -base, pc-fixorgD); fixorgD := pc-1
+    ELSE Put2a(Ldr, RH, -base, pc-fixorgD); fixorgD := pc-1
     END
   END GetSB;
 
@@ -772,11 +782,7 @@
   BEGIN (*x.type.form = ORB.Proc*)
     IF x.mode = ORB.Const THEN
       IF x.r >= 0 THEN Put3(BL, 7, (x.a DIV 4)-pc-1)
-      ELSE (*imported*)
-        IF pc - fixorgP < 1000H THEN
-          Put3(BL, 7, ((-x.r) * 100H + x.a) * 1000H + pc-fixorgP); fixorgP := pc-1
-        ELSE ORS.Mark("fixup impossible")
-        END
+      ELSE (*imported*) Put3a(BL, -x.r, x.a, pc-fixorgP); fixorgP := pc-1
       END
     ELSE
       IF x.mode <= ORB.Par THEN load(x); DEC(RH)
```

**Modules.Mod**

```diff
--- FPGAOberon2013/Modules.Mod  2020-02-26 01:15:33.000000000 +0100
+++ Oberon-allow-importing-any-number-of-modules/Sources/FPGAOberon2013/Modules.Mod  2020-03-12 18:49:05.000000000 +0100
@@ -1,4 +1,4 @@
-MODULE Modules;  (*Link and load on RISC; NW 20.10.2013 / 8.1.2019*)
+MODULE Modules;  (*Link and load on RISC; NW 20.10.2013 / 8.1.2019 / AP 12.3.20*)
   IMPORT SYSTEM, Files;
   CONST versionkey = 1X; MT = 12; DescSize = 80;
 
@@ -55,7 +55,7 @@
       disp, adr, inst, pno, vno, dest, offset: INTEGER;
       name1, impname: ModuleName;
       F: Files.File; R: Files.Rider;
-      import: ARRAY 16 OF Module;
+      import: ARRAY 64 OF Module;
   BEGIN mod := root; error(0, name); nofimps := 0;
     WHILE (mod # NIL) & (name # mod.name) DO mod := mod.next END ;
     IF mod = NIL THEN (*load*)
@@ -135,9 +135,9 @@
         adr := mod.code + fixorgP*4;
         WHILE adr # mod.code DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV 100000H MOD 10H;
-          pno := inst DIV 1000H MOD 100H;
-          disp := inst MOD 1000H;
+          mno := inst DIV 400000H MOD 40H;
+          pno := inst DIV 4000H MOD 100H;
+          disp := inst MOD 4000H;
           SYSTEM.GET(mod.imp + (mno-1)*4, impmod);
           SYSTEM.GET(impmod.ent + pno*4, dest); dest := dest + impmod.code;
           offset := (dest - adr - 4) DIV 4;
@@ -148,8 +148,8 @@
         adr := mod.code + fixorgD*4;
         WHILE adr # mod.code DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV 100000H MOD 10H;
-          disp := inst MOD 1000H;
+          mno := inst DIV 40000H MOD 40H;
+          disp := inst MOD 40000H;
           IF mno = 0 THEN (*global*)
             SYSTEM.PUT(adr, (inst DIV 1000000H * 10H + MT) * 100000H + mod.num * 4)
           ELSE (*import*)
@@ -166,7 +166,7 @@
         adr := mod.data + fixorgT*4;
         WHILE adr # mod.data DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV 1000000H MOD 10H;
+          mno := inst DIV 1000000H MOD 40H;
           vno := inst DIV 1000H MOD 1000H;
           disp := inst MOD 1000H;
           IF mno = 0 THEN (*global*) inst := mod.data + vno
```

**ORL.Mod**

```diff
--- FPGAOberon2013/ORL.Mod  2020-03-12 19:02:26.000000000 +0100
+++ Oberon-allow-importing-any-number-of-modules/Sources/FPGAOberon2013/ORL.Mod  2020-03-12 18:54:12.000000000 +0100
@@ -67,7 +67,7 @@
       disp, adr, inst, pno, vno, dest, offset: INTEGER;
       name1, impname: ModuleName;
       F: Files.File; R: Files.Rider;
-      import: ARRAY 16 OF Module;
+      import: ARRAY 64 OF Module;
   BEGIN mod := root; error(noerr, name); nofimps := 0;
     WHILE (mod # NIL) & (name # mod.name) DO mod := mod.next END ;
     IF mod = NIL THEN (*link*)
@@ -144,9 +144,9 @@
         adr := mod.code + fixorgP*4;
         WHILE adr # mod.code DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV C20 MOD C4;
-          pno := inst DIV C12 MOD C8;
-          disp := inst MOD C12;
+          mno := inst DIV C22 MOD C6;
+          pno := inst DIV C14 MOD C8;
+          disp := inst MOD C14;
           SYSTEM.GET(mod.imp + (mno-1)*4, impmod);
           SYSTEM.GET(impmod.ent + pno*4, dest); dest := dest + impmod.code;
           offset := (dest - adr - 4) DIV 4;
@@ -157,8 +157,8 @@
         adr := mod.code + fixorgD*4;
         WHILE adr # mod.code DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV C20 MOD C4;
-          disp := inst MOD C12;
+          mno := inst DIV C18 MOD C6;
+          disp := inst MOD C18;
           IF mno = 0 THEN (*global*)
             SYSTEM.PUT(adr, (inst DIV C24 * C4 + MT) * C20 + mod.num * 4)
           ELSE (*import*)
@@ -175,7 +175,7 @@
         adr := mod.data + fixorgT*4;
         WHILE adr # mod.data DO
           SYSTEM.GET(adr, inst);
-          mno := inst DIV C24 MOD C4;
+          mno := inst DIV C24 MOD C6;
           vno := inst DIV C12 MOD C12;
           disp := inst MOD C12;
           IF mno = 0 THEN (*global*) inst := mod.data - Start + vno
```
