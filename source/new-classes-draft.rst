Draft for new classes for temperature control in the next version
=================================================================
Draft for new classes in AVR code

.. uml::

    package sensors{
        class fridgeSensor{
            +temp
            +filters
            +setpoint
        }
        class beer1Sensor{
            +temp
            +filters
            +setpoint
        }
        class beer2Sensor{
            +temp
            +filters
            +setpoint
        }
    }

    package controllers{
        class tempToPidTemp1{
            -"Kp,Ki,Kd"
            +getTemp()
            +getTempDiff()
        }
        class tempToPidTemp2{
            -"Kp,Ki,Kd"
            +getTemp()
            +getTempDiff()
        }
        class tempToPredictiveOnOff1{
            -OvershootEstimator
        }
        class tempToPredictiveOnOff2{
            -OvershootEstimator
        }
        class tempDiffToPwmVal1{
            -degreesInFullRange
        }
        class tempDiffToPwmVal2{
            -degreesInFullRange
        }
        class setPointController{
            -type<<min,max,avg>>
        }
    }

    package actuators{
        class chamberCooler{
        }
        class chamberHeater{
        }
        class directBeerHeater1{
        }
        class directBeerHeater2{
        }
    }

    tempToPidTemp1->tempDiffToPwmVal1:tempDiff
    tempToPidTemp2->tempDiffToPwmVal2:tempDiff

    tempDiffToPwmVal1->directBeerHeater1:pwm
    tempDiffToPwmVal2->directBeerHeater2:pwm

    beer1Sensor-->tempToPidTemp1:filters,setpoint
    beer2Sensor-->tempToPidTemp2:filters,setpoint

    tempToPidTemp1-->setPointController:temp
    tempToPidTemp2-->setPointController:temp

    setPointController-->tempToPredictiveOnOff1:temp
    setPointController-->tempToPredictiveOnOff2:temp

    tempToPredictiveOnOff1-->chamberHeater
    tempToPredictiveOnOff2-->chamberCooler

    fridgeSensor-->setPointController