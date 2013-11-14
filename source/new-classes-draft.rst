Draft for new classes for temperature control in the next version
=================================================================
Draft for new classes in AVR code

.. uml::

    class Ticks{
        +seconds()
        +minutes()
        +hours()
        +ticks() {return seconds();}
    }

    abstract Readable<T> {
        T read()
    }
    
    abstract Writable<T> {
        write(T t)
    }
    
    CachedReadable <|-- Readable
    
    abstract CachedReadable<T> {
       cached value which is automatically invalidated at every tick
       -char updateTick;
       # T cachedValue
       # compute()
       # isUpToDate() { return updateTick == Ticks.ticks(); }

       +update() { updateTick = Ticks.ticks(); compute();}
       +read() { if(!isUpToDate()){update()}; return cachedValue;}
       +invalidate(){ updateTick = -1; }
    }

    abstract TickedCachedReadable<T> {
        -const char ticksValid;
        isUpToDate() { return (ticksSince(updateTick) < ticksValid)}
    }

    TickedCachedReadable<|--CachedReadable
    
    Actuator <|-- Writable
    abstract Actuator {
    }
    
    CachedWritable <|-- Writable
    CachedWritable <|-- Readable
    abstract class CachedWritable<T> {
       remembers the last value set so it can be queried/logged
       write(T) target.write(T); lastwrite = T;
       read() lastwrite
       Writable<T> target
       T lastwrite
    }
    
    BasicTempSensor <|-- CachedReadable
    class BasicTempSensor {
       compute()
    }

    class FilteredTempSensor {
        CascadedFilter fastFiltered
        CascadedFilter slowFiltered
        detectPosPeak()
        detectNegPeak()
    }
    FilteredTempSensor<|--BasicTempSensor

    class SlopeFilteredTempSensor {
        CascadedFilter slopeFiltered
    }
    SlopeFilteredTempSensor<|--FilteredTempSensor
    
    ContainedCachedReadable <|-- CachedReadable

    abstract class ContainedCachedReadable<T> {
       # container
       delegates compute() function of value to container

       compute(){ container.compute()}
    }
       
    abstract class ReadableContainer  {
       compute() recomputes if needed, sets value on all contained values
       reset() flags a recompute on next call of compute()
       # ContainedCachedReadable[] values
       - bool recompute   
    }
    
    PIDController <|-- ReadableContainer
    
    class PIDController {
        Readable kp, ki, kd, SV, PV, ..etc.
        ContainedCachedReadable  p, i, d, tempDiff
        compute() uses input values and updates the contained computed values
    }


Diagram below is invalid and should be converted to an object diagram.


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
        class tempDiffToPwmVal3{
            -degreesInFullRange
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
    setPointController-->tempDiffToPwmVal3:tempdiff

    tempDiffToPwmVal3-->chamberHeater
    tempToPredictiveOnOff1-->chamberCooler

    fridgeSensor-->setPointController