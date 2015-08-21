# UramuEdit
Danmaku behaviour Visual-Based-Editor


## Pre-Word
The following is a rough mockup of a very very incomplete project, *please* scrutinize with care...

## Uramu-Editor
The Editor functions as a visual designer of the behaviours of projectiles and enemies within danmaku/bullet-hell styled games.

The Editor is not intended to be a fully fledged designer of danmaku games, instead it serves to provide a quick way to put together the AI of entities in a cross-framework implementation through the use of behaviors and interpreters.

Please forward any feature requests / queries to [URL HERE]


# Usage
# Behaviors & Standard Interpreter

Behaviors currently are standard 'dumps' of an array of events. (WIP formatting rules as Editor is designed)

```lua
{{t=3,eventName="addForce",args={x=100,y=0}},{t=5,eventName="addForce",args={x=0,y=100}}}
```

The previous is a simple behavior that has two events at times 3s and 5s.

The Standard Interpreter simply binds or wraps the Event Name with a function - no special extras.

in code form we can assume the following project (LOVE2D) :

The following is Standard Interpreter and Behavior Object implementations packaged with UramuEdit :

[interpreter/Standard.lua]
```lua
local interpreter = {}

local Behaves = require("behavioral/Standard")

function interpreter:new()
 local o = {
  name="Standard",
 } -- create a copy of the base data...
 setmetatable(o, self)
 self.__index = self
 return o
end

--[[
  Standardised Interpreter, simply wraps/binds to behavior object from script.`
--]]

function interpreter:readFor(behavior,wrappedEvents,object)
  --Designer can code specially here,`
  --by default this simply passes to the behavior object the eventData and function to bind
  local newBehaviorObject = Behaves:new()
  for k,v in pairs(behavior) do
    --will only add events interpreter reads for - any unknowns are discarded
    if(wrappedEvents[v.eventName]) then
      newBehaviorObject:addEvent(v,wrappedEvents[v.eventName],object)
    end
  end
  return newBehaviorObject
end

return interpreter
```


[behavioral/Standard.lua]
```lua
local behaves = {
}

function behaves:new()
 local o = {
  name="Standard",
  Time = 0,
  TimeOffset = 0,
  events = {},
 } -- create a copy of the base data...
 setmetatable(o, self)
 self.__index = self
 return o
end

--Behavior object for Standard Interpreter.

function behaves:setTime(t)
  self.Time = t
end

function behaves:setTimeOffset(t)
  self.TimeOffset = t
end

function behaves:update(dt)
  self.Time = self.Time+dt
  for k,v in pairs(self.events) do
    if(self.Time >=  v.at + self.TimeOffset and not v.state) then --we hit a timed event?
      v.state = not v.state
      if(not v.objectSpecific) then
        v.called(v.args)
      else
        v.called(v.objectSpecific,v.args)
      end
    end
  end
end

function behaves:addEvent(eventData,f,objectSpecific)
  self.events[#self.events + 1] = {}
  self.events[#self.events].objectSpecific = objectSpecific
  self.events[#self.events].at = eventData.t
  self.events[#self.events].args = eventData.args
  self.events[#self.events].called = f
end

return behaves
```

The following is 'user' code :

[main.lua]
```lua
local EntityImplement = require("Entity.lua")
--For sake of simplicity the Behavior script is assumed to be a file whos contents is read, here it is a string we will use.

local BehaviorScript = "return {{t=3,eventName="addForce",args={x=100,y=0}},{t=5,eventName="addForce",args={x=0,y=100}}}"


function love.load()
  MyEntity = EntityImplement:new()
  MyEntity:attachBehavior(BehaviorScript)
end

function love.update(dt)
 MyEntity:update(dt) --Update both Entity and internally its behavior, dt is important as it controls the timings internally.

end

function love.draw()
 MyEntity:draw()

end
```


[Entity.lua]
```lua
local Entity = {
}

local Interperter = require("interpreter/Standard")

function Entity:new()
 local o = {
  position = {0,0},
  deltaPosition = {0,0},
  color = {0,255,0,255*0.80},
  behavior = nil,
 } -- create a copy of the base data...
 setmetatable(o, self)
 self.__index = self
 return o
end

--[[
  Implementation of a Entity that uses Standard Interpreter
--]]

--Attaches a new behavior object using the passed behavior.
function Entity:attachBehavior(behaviorScript)
  local buffer = loadstring(behaviorScript)() --The behavior script exists as a string, run it to fetch the array
  self.behavior = Interperter:readFor(buffer,self.Interprets,self)
end

function Entity:setPos(x,y)
 self.position = {x or self.position[1],y or self.position[2]}
end

function Entity:updateForces(dt)
  deltaPosition = self.deltaPosition
  position = self.position
  local dtxa = dt*deltaPosition[1]
  deltaPosition[1] = deltaPosition[1]-dtxa
  local dtya = dt*deltaPosition[2]
  deltaPosition[2] = deltaPosition[2]-dtya
  
  position[1] = position[1]+dtxa
  position[2] = position[2]+dtya
end

function Entity:update(dt)
  self:updateForces(dt)
  self.behavior:update(dt)
end

function Entity:addForce(force_x,y)
  local xForce,yForce = 0,0
  if(type(force_x) == "table") then 
    xForce,yForce = force_x[1] or force_x.x or 0, force_x[2] or force_x.y or 0
  elseif(type(force_x) == "number" or force_x == nil ) then
    xForce,yForce = force_x or 0,y or 0
  end
  --Supports independant nils/number and table supports t.x,t.y or index [1],[2]
  self.deltaPosition[1] = self.deltaPosition[1] + xForce
  self.deltaPosition[2] = self.deltaPosition[2] + yForce
end

function Entity:draw()
 love.graphics.setColor(self.color)
 love.graphics.circle("fill",self.position[1],self.position[2],6,16)
 love.graphics.setColor(255,255,255,255)
end

Entity.Interprets = {
  addForce = Entity.addForce,
}
return Entity
```


In the above project, we create a Entity implementation that makes use of the behavior implementation.
The Entity.Inperprets table contains a list of members whos keys are the event name to bind and value is the function to call in place
Here the simple addForce event calls Entity.addForce passing arguments.

Lets break down the Interpreter and Behavior code :

```lua
interpreter:readFor(behavior,wrappedEvents,object)
```
Taking the array passed in @behavior (in future handling files and strings aswell) and the wrappedEvents array the Interpreter will bind the matching events.
The third argument is for OOP implementations like the above Entity, here the OOP objects 'self' argument is used.

The readFor method returns a behavior object that tracks the internal events.
The behavior object can be updated independantly with its own :update(dt) method - A dt of 0 would halt the internal lifetime.

//

The above is subject to change at any time.

More details on the actual Editor Application to come.
