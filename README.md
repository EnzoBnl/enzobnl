# MarkDown titles numerator
[![Actions Status](https://github.com/enzobnl/md-titles-numerator/workflows/test/badge.svg)](https://github.com/enzobnl/pycout/actions)

[![Actions Status[enzobnl.github.io](https://github.com/enzobnl/md-titles-numerator/workflows/PyPI/badge.svg)](https://github.com/enzobnl/pycout/actions)

Auto numerate markdown files titles, for example

```markdown
# Some title
## Some subtitle
```

will be numerated numerated as follow

```markdown
# I/ Some title
## 1) Some subtitle
```

## Install
`pip install mdtitlesnumerator`

## Usage
```python
from mdtitlesnumerator import numerate

with open("/path/file.md", "r") as input_file:
    print("numerated content:")
    print(numerate(input_file))

```enzobnl.github.io)
See you there !
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzA0Mzg3OTA2XX0=
-->