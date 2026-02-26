
## PROGRAM Lab_01_AAut

### Header

```iecst

PROGRAM Lab_01_AAut
VAR
   
   fbAlarmSensor        : FB_HEVS_SetAlarm(diAlarmId := 51);
   fbWarningSensor      : FB_HEVS_SetWarning(diWarningId := 52);
   
   
   dmO300_DL_optical    : DM_O300_DL_optical(strName := 'O300.DL-11199079');
   uaO300_DL_TestStruct : UA_O300_DL;
   
   // Some variable to test the FB.
   xSensorEnable        : BOOL;
   strNameOfFb          : STRING;
   xA                   : BOOL;
   xQ                   : BOOL;
   lrDistance           : LREAL;
   
   uliLoop              : ULINT;
   xInit                : BOOL;
   
   // Tempo variables for Signal generator
   myRadian             : LREAL;
   uiInterface          : ST_Lab_01_Interface;
   xSquarePositive      : BOOL;
   uliModLoop           : ULINT;
   
   diCheckRadian        : DINT;
END_VAR
VAR CONSTANT
   PI                   : LREAL := 3.14159265358979323846;
   SAMPLE_TIME          : LREAL := 0.01;
END_VAR

```

### Core

```iecst
uliLoop := uliLoop + 1;
IF NOT xInit THEN
   xInit := TRUE;
   uiInterface.lrAmplitude := 1;
   uiInterface.lrFrequency := 1;
   uiInterface.xFalseSinTrueSquare := FALSE;
END_IF

IF uiInterface.lrFrequency <> 0 THEN
   myRadian := ULINT_TO_LREAL(uliLoop) * SAMPLE_TIME * uiInterface.lrFrequency * (2*PI);
END_IF;
uliModLoop :=  uliLoop MOD 100;

diCheckRadian := LREAL_TO_DINT(myRadian) MOD 628;

xSquarePositive := (uliModLoop < 50);

IF NOT uiInterface.xFalseSinTrueSquare THEN
   uiInterface.lrMySignal := uiInterface.lrAmplitude * SIN(myRadian);
ELSE
   IF xSquarePositive THEN
      uiInterface.lrMySignal :=  uiInterface.lrAmplitude ;
   ELSE
      uiInterface.lrMySignal :=  uiInterface.lrAmplitude * -1;
   END_IF
END_IF

(*
   First part of the lab
*)


(*
   Second part of the lab
   Test the Function Block
*)
dmO300_DL_optical.mSensorEnable(enable := xSensorEnable);

// For tests, use this line (else, the next)
// dmO300_DL_optical(hw := uaO300_DL_TestStruct);
dmO300_DL_optical(hw := GVL_Abox.uaAboxInterface.uaO300_DL_Optic);

strNameOfFb := dmO300_DL_optical.GetName;
xA := dmO300_DL_optical.GetAlarmBit;
xQ := dmO300_DL_optical.GetQualityBit;
lrDistance := dmO300_DL_optical.GetDistance;

(*
   Call some alarms and warnings for test
*)

fbAlarmSensor(// Bit activation of Alarm and Ack
              xSetAlarm := dmO300_DL_optical.GetAlarmBit,
              xAckAlarmTrig := TRUE,
              // Alarm Parameters
              Value := 1,
              Message := CONCAT('Abort Because of Sensor ', dmO300_DL_optical.GetName), 
              Category := E_EventCategory.Abort,
              // Reference to plc time from PackTag
              plcDateTimePack   := PackTag.Admin.PLCDateTime,
              // Link to PackTag Admin
              stAdminAlarm := PackTag.Admin.Alarm,
              stAdminAlarmHistory := PackTag.Admin.AlarmHistory);
         
fbWarningSensor(xSetWarning := dmO300_DL_optical.GetQualityBit,
                xAckWarningTrig := TRUE,
                // Warning Parameters
                Value := 1,
                Message := CONCAT('Quality bit for Sensor ', dmO300_DL_optical.GetName),
                Category := E_EventCategory.Warning,
                // Reference to plc time from PackTag
                plcDateTimePack   := PackTag.Admin.PLCDateTime,
                // Link to PackTag Admin
                stAdminWarning := PackTag.Admin.Warning);

// End of file
```

---

## FUNCTION_BLOCK CM_Gripper

### Header

```iecst
FUNCTION_BLOCK CM_Gripper EXTENDS CM_Abstract
VAR_INPUT
END_VAR
VAR_OUTPUT
END_VAR
VAR
   uliCmGripperLoop    : ULINT;
   xInit               : BOOL;
   
   (*
      List of devices used in this Module
   *)
   fbI_Gripper         : FB_I_Gripper;
   // dmO300_DL_optical      : DM_O300_DL_optical

   (*
      List of alarms and warning with unique ID
   *)   
   fbAlarmIgripper     : FB_HEVS_SetAlarm(diAlarmId := UDINT_TO_DINT(THIS^.uiUniqueId + 1));
   fbSetAlarm_Sensor   : FB_HEVS_SetAlarm(diAlarmId := UDINT_TO_DINT(THIS^.uiUniqueId + 2));
   fbSetWarningSensor  : FB_HEVS_SetWarning(diWarningId := UDINT_TO_DINT(THIS^.uiUniqueId + 1));

   uiSensorDistance    : LREAL;
   
   xTestEnableGripper  : BOOL;
   xTestDisableGripper : BOOL;
   xTestOpenGripper    : BOOL;
   xTestCloseGripper   : BOOL;
   
END_VAR

```

### Core

```iecst
uliCmGripperLoop := uliCmGripperLoop + 1;
IF NOT xInit THEN
   xInit := TRUE;
END_IF

(* Call to parent *)
SUPER^(Status_ModeCurrent := Status_ModeCurrent,
       Status_StateCurrent := Status_StateCurrent);

(*
   Call devices
*)
fbI_Gripper(hw_Festo := GVL_Abox.uaAboxInterface.uaSchunkGripper,
           hw_Schunk := GVL_Abox.uaAboxInterface.uaSchunk);
         
         
IF xTestEnableGripper THEN
   fbI_Gripper.M_EnableDevice(Enable := TRUE);
   IF fbI_Gripper.InOp THEN
      xTestEnableGripper := FALSE;
   END_IF
END_IF

IF xTestDisableGripper THEN
   fbI_Gripper.M_EnableDevice(Enable := FALSE);
   IF NOT fbI_Gripper.InOp THEN
      xTestDisableGripper := FALSE;
   END_IF
END_IF

IF xTestOpenGripper THEN
   fbI_Gripper.M_SetOpen(rLimitOpen_mm := 1,
                        udiTimeOut_ms := 450);
   IF fbI_Gripper.IsOpen THEN
      xTestOpenGripper := FALSE;
   END_IF
END_IF         

IF xTestCloseGripper THEN
   fbI_Gripper.M_SetClose(rLimitClosed_mm := 9,
                         udiTimeOut_ms := 450);
   IF fbI_Gripper.IsClosed THEN
      xTestCloseGripper := FALSE;
   END_IF
END_IF         


(*
   Alarms and warnings here
*)
fbAlarmIgripper(xSetAlarm := fbI_Gripper.Error,
                xAckAlarmTrig := FC_HEVS_GetAckAlarmById(fbAlarmIgripper.GetUniqueId),
                xAckAlarmTrig := TRUE,
                Value := 1,
                Message := CONCAT('Gripper: ', fbI_Gripper.ErrorId),
                Category := E_EventCategory.Abort,
                // Reference to plc time from PackTag
                plcDateTimePack   := PackTag.Admin.PLCDateTime,
                // Link to PackTag Admin
                stAdminAlarm := PackTag.Admin.Alarm,
                stAdminAlarmHistory := PackTag.Admin.AlarmHistory);                  


(*   End of Control Modules *)         

```

<!--End of the file-->