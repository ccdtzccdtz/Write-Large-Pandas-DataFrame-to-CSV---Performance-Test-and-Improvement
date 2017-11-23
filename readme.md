
# Write Large Pandas DataFrame to CSV - Performance Test and Improvement

The `pd.to_csv` function is a common way to conveniently write dataframe content to txt file such as csv. It is normally very efficient but will suffer slowness when handling large dataframe. Some alternative code pieces are introduced in this test and compared with the default `pd.to_csv` performance.

## Load Libraries


```python
import pandas as pd
import numpy as np
import csv
import copy
%load_ext Cython
```

    The Cython extension is already loaded. To reload it, use:
      %reload_ext Cython
    

## Create Dataframe


```python
df = pd.DataFrame(np.random.randn(100000,20))
df.head()
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>10</th>
      <th>11</th>
      <th>12</th>
      <th>13</th>
      <th>14</th>
      <th>15</th>
      <th>16</th>
      <th>17</th>
      <th>18</th>
      <th>19</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>-0.906642</td>
      <td>-0.861386</td>
      <td>0.614656</td>
      <td>-0.907063</td>
      <td>0.594934</td>
      <td>-0.207979</td>
      <td>3.481780</td>
      <td>0.819977</td>
      <td>-0.565264</td>
      <td>-0.332090</td>
      <td>-0.784525</td>
      <td>-0.293207</td>
      <td>1.224856</td>
      <td>-2.319758</td>
      <td>1.178547</td>
      <td>-0.515090</td>
      <td>0.709837</td>
      <td>-1.091345</td>
      <td>-0.039718</td>
      <td>2.091454</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2.613523</td>
      <td>0.785575</td>
      <td>-1.775879</td>
      <td>-0.459838</td>
      <td>1.902544</td>
      <td>0.509347</td>
      <td>0.020503</td>
      <td>-1.791932</td>
      <td>0.647267</td>
      <td>-0.934782</td>
      <td>-0.677657</td>
      <td>0.492583</td>
      <td>-1.002009</td>
      <td>-0.730491</td>
      <td>-0.624470</td>
      <td>-2.488585</td>
      <td>1.016912</td>
      <td>0.906007</td>
      <td>0.658830</td>
      <td>-0.264958</td>
    </tr>
    <tr>
      <th>2</th>
      <td>-0.036129</td>
      <td>-1.746564</td>
      <td>-0.140750</td>
      <td>0.663674</td>
      <td>0.094645</td>
      <td>0.049312</td>
      <td>-0.632624</td>
      <td>-0.550121</td>
      <td>0.395835</td>
      <td>-1.423970</td>
      <td>-0.521202</td>
      <td>-0.126024</td>
      <td>0.229703</td>
      <td>1.164841</td>
      <td>0.697109</td>
      <td>0.917896</td>
      <td>0.880719</td>
      <td>1.236428</td>
      <td>0.837311</td>
      <td>0.079312</td>
    </tr>
    <tr>
      <th>3</th>
      <td>-2.400462</td>
      <td>-0.155776</td>
      <td>-0.130211</td>
      <td>1.515387</td>
      <td>-0.157206</td>
      <td>1.249488</td>
      <td>-0.328810</td>
      <td>-0.701600</td>
      <td>-0.961956</td>
      <td>-0.752843</td>
      <td>-0.629910</td>
      <td>-0.116808</td>
      <td>0.340790</td>
      <td>0.127246</td>
      <td>-0.761531</td>
      <td>-1.207977</td>
      <td>-0.707612</td>
      <td>-1.860185</td>
      <td>-1.287765</td>
      <td>1.415692</td>
    </tr>
    <tr>
      <th>4</th>
      <td>-0.049455</td>
      <td>2.054475</td>
      <td>-1.809350</td>
      <td>-1.523187</td>
      <td>0.602191</td>
      <td>0.167721</td>
      <td>-0.311740</td>
      <td>0.626712</td>
      <td>0.642302</td>
      <td>0.154328</td>
      <td>0.237725</td>
      <td>0.307409</td>
      <td>1.110008</td>
      <td>-0.419831</td>
      <td>1.120733</td>
      <td>-1.385226</td>
      <td>1.827813</td>
      <td>1.175392</td>
      <td>-0.149792</td>
      <td>-1.231708</td>
    </tr>
  </tbody>
</table>
</div>



## Performance Test

### Use Pandas to_csv


```python
%timeit -r3 df.to_csv('tocsv.csv',index=False,float_format='%10.11f') 
```

    1 loop, best of 3: 2.96 s per loop
    

### Use Numpy savetxt

The idea is to convert dataframe to ndarray and save to txt file


```python
%timeit -r3 np.savetxt("numpytotxt.csv", df.values, delimiter=",",fmt='%10.11f')
```

    1 loop, best of 3: 784 ms per loop
    

### Use one-liner string and combined with Numpy tofile

The idea is to combine all field to a big string column and only write that column to csv.


```python
def oneliner(df,str_format=True):
    columns=list(df)
    if not str_format:
        df=df.astype(str)    
    s=copy.copy(df[columns[0]])
    for col in columns[1:]:
        s+=","+df[col]        
    s.values.tofile('numpytotxt_oneliner.csv',sep='\n')
```


```python
%timeit -r3 oneliner(df,False)
```

    1 loop, best of 3: 2.2 s per loop
    

__This saved csv has quotes in each line. One may need to remove those to get a clean csv.__

### Use one-liner string and combined with Python file.write function

Use the same idea of combing columns to one string columns, and use \n to join them into a large string. Use native python write file function


```python
def df_to_string(df,str_format=True):
    columns=list(df)
    if not str_format:
        df=df.astype(str)
    s=copy.copy(df[columns[0]])
    for col in columns[1:]:
        s+=","+df[col]
    return '\n'.join(s)
def df_to_csv_one(df,str_format=True):
    file='oneliner_write.csv'
    with open(file,"w") as f:
        f.write(df_to_string(df,str_format))
```


```python
%timeit -r3 df_to_csv_one(df,False)
```

    1 loop, best of 3: 2.21 s per loop
    

### Use one-liner string and combined with Cython

same idea with combine everything to a large string and use Cython to write to file


```python
%%cython
from libc.stdio cimport fopen, FILE, fclose, fprintf
def c_write_to_file(filename, content):
    filename_byte_string = filename.encode("UTF-8")
    cdef char* fname = filename_byte_string
    cdef char* line = content
    cdef FILE* cfile
    cfile = fopen(fname, "w")
    if cfile == NULL:
        return
    fprintf(cfile, line)
    fclose(cfile)
    return []

```


```python
def df_to_csv_cython(df,str_format=True):
    c_write_to_file('oneliner_cython.csv', df_to_string(df,str_format).encode("UTF-8"))
```


```python
%timeit -r3 df_to_csv_cython(df,False)
```

    1 loop, best of 3: 2.4 s per loop
    

### Test with Larger (10M,20) Dataframe


```python
df = pd.DataFrame(np.random.randn(10000000,20))
print("Pandas tocsv")
%timeit -r3 -n1 df.to_csv('tocsv.csv',index=False,float_format='%10.11f') 
print("Numpy savetxt")
%timeit -r3 -n1 np.savetxt("numpytotxt.csv", df.values, delimiter=",",fmt='%10.11f')
print("Oneliner with numpy tofile")
%timeit -r3 -n1 oneliner(df,False)
print("Oneliner to string with Pyton f.write")
%timeit -r3 -n1 df_to_csv_one(df,False)
print("Oneliner to string with Cython")
%timeit -r3 -n1 df_to_csv_cython(df,False)
```

    Pandas tocsv
    1 loop, best of 3: 4min 49s per loop
    Numpy savetxt
    1 loop, best of 3: 1min 15s per loop
    Oneliner with numpy tofile
    1 loop, best of 3: 3min 38s per loop
    Oneliner to string with Pyton f.write
    1 loop, best of 3: 3min 55s per loop
    Oneliner to string with Cython
    1 loop, best of 3: 3min 28s per loop
    

### Test with Larger and Narrow(50M,3) Dataframe (string)


```python
df = pd.DataFrame(np.random.randn(50000000,3))
df=df.astype(str)
print("Pandas tocsv")
%timeit -r3 df.to_csv('tocsv.csv',index=False) 
print("Numpy savetxt")
%timeit -r3 np.savetxt("numpytotxt.csv", df.values, delimiter=",",fmt='%s')
print("Oneliner with numpy tofile")
%timeit -r3 oneliner(df)
print("Oneliner to string with Pyton f.write")
%timeit -r3 df_to_csv_one(df)
print("Oneliner to string with Cython")
%timeit -r3 df_to_csv_cython(df)
```

    Pandas tocsv
    1 loop, best of 3: 2min 13s per loop
    Numpy savetxt
    1 loop, best of 3: 1min 30s per loop
    Oneliner with numpy tofile
    1 loop, best of 3: 36.6 s per loop
    Oneliner to string with Pyton f.write
    1 loop, best of 3: 53.4 s per loop
    Oneliner to string with Cython
    1 loop, best of 3: 37.4 s per loop
    

## Performance Summary

* Pandas default to_csv is the slowest in all cases. However, it is the most convenient in terms handling all kinds of special cases such as quotation, missing value, etc. Recommended for general purposes.
* Numpy alternatives are faster but additional cleaning is needed
* When dealing with numbers, all oneliner alternatives are relatively slow because of the overhead of casting numbers to strings
* One liner with Python f.write and Cython is faster with large narrow string dataframe and the performance is comparable

### Conlusion

* Pandas tocsv is recommended for general purpose when datatype is complicated.
* Numpy savetxt has good performance if all data in the dataframe is number.
* Oneliner is good option to deal with dataframe mainly with string.

| This | is   |
|------|------|
|   a  | table|



| Pandas tocsv      | 2.98 s | 4min 49s |2min 13s|
| :------------- | :-------------: | :-----: | :-----: |
| Numpy savetxt      | 770 ms |   1min 15s   |   1min 30s |
| Oneliner with numpy tofile |   2.2 s    |   3min 38s  |36.6 s|
| Oneliner to string with Pyton f.write |   2.21    |    3min 55s |53.4 s|
| Oneliner to string with Cython |   2.4 s    |   3min 28s  |37.4 s|


| Methods       | DataFrame (100000,20)           | DataFrame (10000000,20)  | DataFrame (50000000,3) (String)|
| :------------- |:-------------:|: -----:|: -----:|
| Pandas tocsv      | 2.98 s | 4min 49s |2min 13s|
| Numpy savetxt      | 770 ms |   1min 15s   |   1min 30s |
| Oneliner with numpy tofile |   2.2 s    |   3min 38s  |36.6 s|
| Oneliner to string with Pyton f.write |   2.21    |    3min 55s |53.4 s|
| Oneliner to string with Cython |   2.4 s    |   3min 28s  |37.4 s|



## Test Environment

Window 10 Pro 

CPU i7-7700 @ 3.6 GHz

Python 3.6.1

Numpy 1.13.3

Pandas 0.20.1

## Resources

__[Cython with Anaconda on Windows](http://https://github.com/cython/cython/wiki/InstallingOnWindows)__<br>
__[Using Microsoft Visual C with Python](https://matthew-brett.github.io/pydagogue/python_msvc.html)__

