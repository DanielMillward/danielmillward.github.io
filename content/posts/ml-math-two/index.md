+++
date = '2021-03-28'
draft = false
title = 'Machine Learning Math Pt. 2 - More Linear Transformations'
categories = ["Machine Learning Math"]
+++

This is a continuation of [my previous post](/posts/ml-math-one/).

If the two basis vectors for a transformation happen to line up on the same line, that’s called linear dependence, i.e. you can remove either i-hat or j-hat and get the same total span. 

## For solving Ax = b:  

 - $\boldsymbol{b}$ needs to be in the span of $\boldsymbol{A}$ to be solvable.  

 - We can solve for $\boldsymbol{x}$ by using inverse matrices. There’s a bunch of great resources [like this]( https://people.richland.edu/james/lecture/m116/matrices/inverses.html) that’ll help you understand how it works. What’s important is knowing that multiplying an inverse matrix by the original matrix basically nullifies the transformation:

$$
A^{-1}A=I_{n}
$$

 - The number of columns (basis vectors) of $\boldsymbol{A}$ needs to be greater than or equal to its height (the number of dimensions) to have a chance of having a valid solution. This should make sense since if you only have two basis vectors but 3 dimensions, that only has the span of a plane, rather than 3D space that $\boldsymbol{b}$ could exist on.

 - For $\boldsymbol{A}^{-1}$ to exist, $\boldsymbol{A}$ needs to be a square to make the multiplication work. There may still be a solution if $\boldsymbol{A}$ is not a square, but you can’t use matrix inversion.
 
## Norms –

Norms are a way to determine how “long” a vector is. Traditionally, this is done by using the distance equation. But since the start is always the origin, we simplify it to  

$$
x \left \| \vec{\beta} \right \|_2=\sqrt{bet_{0}^{2}+\beta_{1}^{2}}
$$

Where $\beta_{0}$ and $\beta_{1}$ are the coordinates of the vector.
That’s called the L2-Norm. Another way of finding length is just by adding those vector values together -

$$
\left \| \vec{\beta} \right \|_2=|\beta_{0}|+|\beta_{1}|
$$

That’s the L-1 Norm. We can generalize this for all L-whatever norms:

$$
||X||_{p}=(\sum |x_i|^{p})^\frac{1}{p}
$$

It’s common to work with the L2 norm squared, since that’s just $\boldsymbol{X}^{T}\boldsymbol{X}$. Don’t use this for values near zero though, use L1 for that.
The Max-Norm, or L-inf, is just the biggest value in the vector.
To get the “size” of a whole matrix, use the Frobenius norm:

$$
||\boldsymbol{A}||_{p}=\sqrt{\sum_{i,j} \boldsymbol{A}_{i,j}^{2}}
$$
 
## Some facts about diagonal matrices –
The thing about diagonal matrices is that it’s very easy to calculate some of their properties and forms. For example, their inverse is just

$$
[\frac{1}{v}, ... , \frac{1}{v_{n}}]^{T}
$$

And multiplying a matrix by a diagonal matrix is just element-wise multiplication:

$$
diag(\boldsymbol{v})\boldsymbol{x}=\boldsymbol{v} \odot \boldsymbol{x}
$$
 
## Other matrices –
A symmetric matrix is where $\boldsymbol{A}^{T}=\boldsymbol{A}$.
An orthogonal matrix is square, where the rows are “normalized”, i.e. the L-2 norm (the standard) of each column is 1, and the column vectors are orthogonal (90 deg) to each other (Their dot product is 0, while the dot product with themselves is 1). Two fun facts about orthogonal matrices: one, their rows are also orthogonal when you normalize the column vectors, and two, it has these properties:

$$
\boldsymbol{A}^{T}\boldsymbol{A}=\boldsymbol{A}\boldsymbol{A}^{T}=\boldsymbol{I}_{n}
$$

And

$$
\boldsymbol{A}^{-1}=\boldsymbol{A}^{T}
$$

Thus, a very cheap inverse matrix to compute.
 
## Determinant - 
The determinant of a matrix is the scale by which the area found by multiplying i-hat by j-hat (the unit square) is scaled by during the transformation done by the matrix. It can be found by doing this for a 2x2 matrix:

$$
det\left ( \begin{bmatrix}a & b\\\\ c & d\end{bmatrix} \right )=(a+b)(c+d)=ad-bc
$$

If the orientation is flipped then the determinant is negative. If the transformation squishes the space into a lower dimension, the determinant is 0. For the inverse of a matrix to exist, det(A) must not be 0.
 