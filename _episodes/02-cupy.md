---
title: "Using your GPU with CuPy"

teaching: 45

exercises: 15

questions:
- "How can I increase the performance of code that uses NumPy?"
- "How can I copy NumPy arrays to the GPU?"

objectives:
- "Be able to indicate if an array, represented by a variable in an iPython shell, is stored in host or device memory."
- "Be able to copy the contents of this array from host to device memory and vice versa."
- "Be able to select the appropriate function to either convolve an image using either CPU or GPU compute power."
- "Be able to quickly estimate the speed benefits for a simple calculation by moving it from the CPU to the GPU."

keypoints:
- "CuPy provides GPU accelerated version of many NumPy functions."
- "Always have CPU and GPU versions of your code so that you can compare performance, as well as validate your code."
---

# Introduction to CuPy

[CuPy](https://cupy.dev) is a GPU array library that implements a subset of the NumPy and SciPy interfaces.
This makes it a very convenient tool to use the compute power of GPUs for people that have some experience with NumPy, without the need to write code in a GPU programming language such as CUDA, OpenCL, or HIP.

# Convolution in Python

We start by generating an artificial "image" on the host using Python and NumPy; the host is the CPU on the laptop, desktop, or cluster node you are using right now, and from now on we may use *host* to refer to the CPU and *device* to refer to the GPU.
The image will be all zeros, except for isolated pixels with value one, on a regular grid. The plan is to convolve it with a Gaussian and inspect the result.
We will also record the time it takes to execute this convolution on the host.

We can interactively write and executed the code in an iPython shell or a Jupyter notebook.

~~~
import numpy as np

# Construct an image with repeated delta functions
deltas = np.zeros((2048, 2048))
deltas[8::16,8::16] = 1
~~~
{: .language-python}

To get a feeling of how the whole image looks like, we can display the top-left corner of it.

~~~
import pylab as pyl
# Necessary command to render a matplotlib image in a Jupyter notebook.
%matplotlib inline

# Display the image
# You can zoom in using the menu in the window that will appear
pyl.imshow(deltas[0:32, 0:32])
pyl.show()
~~~
{: .language-python}

### Background
The computation we want to perform on this image is a convolution, once on the host and once on the device so we can compare the results and execution times.
In computer vision applications, convolutions are often used to filter images and if you want to know more about them, we encourage you to check out [this github repository](https://github.com/vdumoulin/conv_arithmetic) by Vincent Dumoulin and Francesco Visin with some great animations. We have already seen that we can think of an image as a matrix of color values, when we convolve that image with a particular filter, we generate a new matrix with different color values. An example of convolution can be seen in the figure below (illustration by Michael Plotke, CC BY-SA 3.0, via Wikimedia Commons).

![Example of convolution](../fig/2D_Convolution_Animation.gif)

In our example, we will convolve our image with a 2D Gaussian function shown below:

<!--<div style="font-size:1.5em">-->
$$G(x,y) = \frac{1}{2\pi \sigma^2} \exp\left(-\frac{x^2 + y^2}{2 \sigma^2}\right),$$
<!--</div>-->

where $x$ and $y$ are the "coordinates in our matrix, i.e. our row and columns, and $$\sigma$$ controls the width of the Gaussian distribution. Convolving images with 2D Gaussian functions will change the value of each pixel to be a weighted average of the pixels around it, thereby "smoothing" the image. Convolving images with a Gaussian function reduces the noise in the image, which is often required in [edge-detection](https://en.wikipedia.org/wiki/Gaussian_blur#Edge_detection) since most algorithms to do this are sensitive to noise.

> ## Maps and stencils
> It is often useful to identify the dataflow inherent in a problem. Say, if we want to square a list of numbers, all the operations are independent. The dataflow of a one-to-one operation is called a *map*.
> 
> ![Data flow of a map operation](../fig/mapping.svg){: style="width: 50%"}
> 
> A convolution is slightly more complicated. Here we have a many-to-one data flow, which is also known as a stencil.
> 
> ![Data flow of a stencil operation](../fig/stencil.svg){: style="width: 50%"}
> 
> GPU's are exceptionally well suited to compute algorithms that follow one of these patterns.
{: .callout}

# Convolution on the CPU Using SciPy
Let us first construct the Gaussian, and then display it. Remember that at this point we are still doing everything with standard Python, and not using the GPU yet.

~~~
x, y = np.meshgrid(np.linspace(-2, 2, 15), np.linspace(-2, 2, 15))
dst = np.sqrt(x*x + y*y)
sigma = 1
muu = 0.000
gauss = np.exp(-((dst-muu)**2/(2.0 * sigma**2)))
pyl.imshow(gauss)
pyl.show()
~~~
{: .language-python}

This should show you a symmetrical two-dimensional Gaussian. Now we are ready to do the convolution on the host. We do not have to write this convolution function ourselves, as it is very conveniently provided by SciPy. Let us also record the time it takes to perform this convolution and inspect the top left corner of the convolved image.

~~~
from scipy.signal import convolve2d as convolve2d_cpu

convolved_image_using_CPU = convolve2d_cpu(deltas, gauss)
%timeit convolve2d_cpu(deltas, gauss)
pyl.imshow(convolved_image_using_CPU[0:32, 0:32])
pyl.show()
~~~
{: .language-python}

Obviously, the time to perform this convolution will depend very much on the power of your CPU, but I expect you to find that it takes a couple of seconds.

~~~
1 loop, best of 5: 2.52 s per loop
~~~
{: .output}

When you display the corner of the image, you can see that the "ones" surrounded by zeros have actually been blurred by a Gaussian, so we end up with a regular grid of Gaussians.

# Convolution on the GPU Using CuPy

This is part of a lesson on GPU programming, so let us use the GPU.
Although there is a physical connection - i.e. a cable - between the CPU and the GPU, they do not share the same memory space.
This image depicts the different components of CPU and GPU and how they are connected:

![CPU and GPU are two separate entities, each with its own memory.](../fig/CPU_and_GPU_separated.png)

This means that an array created from e.g. an iPython shell using NumPy is physically located into the main memory of the host, and therefore available for the CPU but not the GPU.
It is not yet present in GPU memory, which means that we need to copy our data, the input image and the convolving function to the GPU, before we can execute any code on it.
In practice, we have the arrays `deltas` and `gauss` in the host's RAM, and we need to copy them to GPU memory using CuPy.

~~~
import cupy as cp

deltas_gpu = cp.asarray(deltas)
gauss_gpu = cp.asarray(gauss)
~~~
{: .language-python}

Now it is time to do the convolution on the GPU.
SciPy does not offer functions that can use the GPU, so we need to import the convolution function from another library, called `cupyx`; `cupyx.scipy` contains a subset of all SciPy routines.
You will see that the GPU convolution function from the `cupyx` library looks very much like the convolution function from SciPy we used previously.
In general, NumPy and CuPy look very similar, as well as the SciPy and `cupyx` libraries, and this is on purpose to facilitate the use of the GPU by programmers that are already familiar with NumPy and SciPy.
Let us again record the time to execute the convolution, so that we can compare it with the time it took on the host.

~~~
from cupyx.scipy.signal import convolve2d as convolve2d_gpu

convolved_image_using_GPU = convolve2d_gpu(deltas_gpu, gauss_gpu)
%timeit convolve2d_gpu(deltas_gpu, gauss_gpu)
~~~
{: .language-python}

Similar to what we had previously on the host, the execution time of the GPU convolution will depend very much on the GPU used, but I expect you to find it will take about 10ms.
This is what I got on a TITAN X (Pascal) GPU:

~~~
1000 loops, best of 5: 20.2 ms per loop
~~~
{: .output}

This is a lot faster than on the host, a performance improvement, or speedup, of 125 times.
Impressive!

> ## Challenge: convolution on the GPU without CuPy 
> 
> Try to convolve the NumPy array `deltas` with the NumPy array `gauss` directly on the GPU, without using CuPy arrays. 
> If this works, it should save us the time and effort of transferring deltas and gauss to the GPU.
>
> > ## Solution
> > 
> > We can directly try to use the GPU convolution function `convolve2d_gpu` with `deltas` and `gauss` as inputs.
> > ~~~
> > convolve2d_gpu(deltas, gauss)
> > ~~~
> > {: .language-python}
> > 
> > However, this gives a long error message ending with:
> > ~~~
> > TypeError: Unsupported type <class 'numpy.ndarray'>
> > ~~~
> > {: .output}
> >
> > It is unfortunately not possible to access NumPy arrays directly from the GPU because they exist in the 
> > Random Access Memory (RAM) of the host and not in GPU memory.
> >
> {: .solution}
{: .challenge}

# Validation

To check that we actually computed the same output on the host and the device we can compare the two output arrays `convolved_image_using_GPU` and `convolved_image_using_CPU`.

~~~
np.allclose(convolved_image_using_GPU, convolved_image_using_CPU)
~~~
{: .language-python}

As you may expect, the result of the comparison is positive, and in fact we computed the same results on the host and the device.

~~~
array(True)
~~~
{: .output}

> ## Challenge: fairer comparison of CPU vs. GPU
> 
> Compute again the speedup achieved using the GPU, but try to take also into account the time spent transferring the data to the GPU and back.
>
> Hint: to copy a CuPy array back to the host (CPU), use the `cp.asnumpy()` function.
>
> > ## Solution
> > 
> > A convenient solution is to group both the transfers, to and from the GPU, and the convolution into a single Python function, and then time its execution, like in the following example.
> > 
> > ~~~
> > def transfer_compute_transferback():
> >     deltas_gpu = cp.asarray(deltas)
> >     gauss_gpu = cp.asarray(gauss)
> >     convolved_image_using_GPU = convolve2d_gpu(deltas_gpu, gauss_gpu)
> >     convolved_image_using_GPU_copied_to_host = cp.asnumpy(convolved_image_using_GPU)
> >    
> > %timeit transfer_compute_transferback()
> > ~~~
> > {: .language-python}
> > ~~~
> > 10 loops, best of 5: 35.1 ms per loop
> > ~~~
> > {: .output}
> >
> > The speedup taking into account the data transfers decreased from 125 (2520 ms/20.2 ms) to 72 (2520 ms/35.1 ms).
> > Taking into account the necessary data transfers when computing the speedup is a better, and more fair, way to compare performance.
> {: .solution}
{: .challenge}

# A shortcut: performing NumPy routines on the GPU.

We saw above that we cannot execute routines from the `cupyx` library directly on NumPy arrays.
In fact we need to first transfer the data from host to device memory.
Vice versa, if we try to execute a regular SciPy routine (i.e. designed to run the CPU) on a CuPy array, we will also encounter an error.
Try the following:

~~~
convolve2d_cpu(deltas_gpu, gauss_gpu)
~~~
{: .language-python}

This results in 
~~~
......
......
......
TypeError: Implicit conversion to a NumPy array is not allowed. Please use `.get()` to construct a NumPy array explicitly.
~~~
{: .output}

So SciPy routines cannot have CuPy arrays as input.
We can, however, execute a simpler command that does not require SciPy.
Instead of 2D convolution, we can do 1D convolution.
For that we can use a NumPy routine instead of a SciPy routine.
The `convolve` routine from NumPy performs linear (1D) convolution.
To generate some input for a linear convolution, we can flatten our image from 2D to 1D (using `ravel()`), but we also need a 1D kernel.
For the latter we will take the diagonal elements of our 2D Gaussian kernel.
Try the following three instructions for linear convolution on the CPU:

~~~
deltas_1d = deltas.ravel()
gauss_1d = gauss.diagonal()
%timeit np.convolve(deltas_1d, gauss_1d)
~~~
{: .language-python}

You could arrive at something similar to this timing result:
~~~
104 ms ± 32.9 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
~~~
{: .output}

We have performed a regular linear convolution using our CPU.
Now let us try something bold.
We will transfer the 1D arrays to the GPU and use the NumPy routine to do the convolution.
Again, we have to issue three commands:

~~~
deltas_1d_gpu = cp.asarray(deltas_1d)
gauss_1d_gpu = cp.asarray(gauss_1d)
%timeit np.convolve(deltas_1d_gpu, gauss_1d_gpu)
~~~
{: .language-python}

You may be surprised that we can issue these commands without error.
Contrary to SciPy routines, NumPy accepts CuPy arrays, i.e. arrays that exist in GPU memory, as input.
[Here](https://docs.cupy.dev/en/v8.2.0/reference/interoperability.html#numpy) you can find some background on why NumPy routines can handle CuPy arrays. 

Also, remember the `np.allclose` command above?
With a NumPy and a CuPy array as input.
That worked for the same reason.

The linear convolution is actually performed on the GPU, which is shown by a nice speedup:

~~~
731 µs ± 106 µs per loop (mean ± std. dev. of 7 runs, 1 loop each)
~~~
{: .output}

So this implies a speedup of a factor 104/0.731 = 142. Impressive.

{% include links.md %}
