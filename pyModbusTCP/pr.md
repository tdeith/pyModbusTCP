Use case: TCP-Modbus protocol allows updating multiple registers via function 16 (0x10), and most reasonable modbus clients will be using this function to write multiple-word data types - EG floating point, system-native integers, unix timestamps... Things where the data is fundamentally incomplete without taking in the whole data-change at once. 

Problem: To do any complex data flows using this project, you need to create your own DataBank subclass. Using this subclass, the obvious way to intercept incoming data is via overriding `on_holding_registers_change`. `on_holding_registers_change` only provides one register at a time, so is not suited to multi-word data-types like floats or timestamps. 

Current workaround: Since we need access to the whole word at once, we need to override `set_holding_registers` instead, like so:
```python
import struct

def twoWordsToFloat(words: list[int]) -> float:
    tempStruct = struct.pack('>HH', words[1], words[0])
    return struct.unpack('>f', tempStruct)[0]

class DataBinding:
    def __init__(self):
        self.binding_width = 2 # This data binding will expect 32-bit floats, so 2 words.

    def doThingWithData(data: list[int]) -> float:
        if len(data) != self.binding_width:
            raise ValueError
        return twoWordsToFloat(data)

class NiceDataBank(DataBank):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.__bindings: dict[DataBinding] = ... # Whatever

    def set_input_registers(self, address: int, word_list: list[int]) -> bool | None:
        count: int = len(word_list)
        if not super().set_input_registers(address, word_list): # Make sure the underlying function works...
            return                                              # If not, *must* return None to satisfy parent class

        word_list = [int(w) & 0xffff for w in word_list] # Only necessary because of the `set_input_registers` override

        while (word_list):                     # Loop until we're out of data which may be bound against
            if address not in self.__bindings: # Make sure there's actually a binding against this address
                word_list = word_list[1:]      # If not: pop off the front of the word list...
                address = address + 1          # ... And see if the next bound address wants any remaining data
                continue

            binding: DataBinding = self.__bindings[address]
            if len(word_list) < binding.width: # Check that this binding is satisfied with this amount of data
                word_list = word_list[1:]      # If not: pop off the front of the word list...
                address = address + 1          # ... And see if the next bound address wants any remaining data
                continue
            
            binding.doThingWithData(word_list[:binding.width]) # Consume the expected words
            word_list = word_list[binding.width:]              # Remove consumed words

        return True # *Must* return True to satisfy parent class
```


Current: need to 

```python

set_holding_registers
```