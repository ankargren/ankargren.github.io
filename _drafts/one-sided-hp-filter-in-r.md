---
layout: post
title: One-Sided HP Filtering in R
category: R

excerpt: Implementation of the one-sided HP filter in R, with an application to the credit gap. 

---

The Hodrick-Prescott filter (CITE!!!!!) is a commonly used filtering technique used in applied macroeconomics. The filtered series is the solution to the minimization problem:
\begin{equation}
x+y=z^2
\end{equation}
where \\(\lambda \\) is a smoothing parameter. The standard (two-sided) HP filter has the inherent drawback that it uses the entire time series available in its computation. This means that values may change every time a new observation is added. If we let the unobserved trend be denoted by \\(\tau_t\\), then the (sometimes) undesired changing behavior of the HP filter is that the trend estimates are conditional expectations of the form \\(\hat\tau_{t|T}\equiv E(\tau_t|y_1, y_2, \dots, y_T)\\). Thus, when new observations arrive and the sample is extended, these conditional expectations are subject to change. 

However, the conditional expectation \\(E(\tau_t\|y_1, y_2, ..., y_T)\\) is precisely the smoothed estimate of \\(\tau_t\\), using Kalman filter nomenclature. As discussed by Stock and Watson (1999), this also means that there, within the Kalman filter framework, is a natural contender for situations when revised trend estimates are undesired: namely the filtered estimates \\(\hat\tau_{t\|t}\equiv E(\tau_t\|y_1, y_2, \dots, y_t) \\). The question is then: can the HP filter be formulated as a state-space model?

The answer is yes (again, see Stock and Watson, 1999). It is written as:
\begin{equation}
y_t=\tau_t+\epsilon_t\\
(1-L)^2 \tau_t=\eta_t
\end{equation}
where
\begin{equation}
E\begin{pmatrix}\epsilon_t \\ \eta_t\end{pmatrix}=\begin{pmatrix}0 \\ 0\end{pmatrix},  V\begin{pmatrix}\epsilon_t \\ \eta_t\end{pmatrix}=\begin{pmatrix}1 & 0 \\ 0 & q
\end{pmatrix}
\end{equation}
and \\(q=\lambda^{-1}\\). Thus, we may easily obtain both filtered (one-sided) and smoothed (two-sided) HP filtered time series by applying the Kalman filter to the state-space model defined above. 
{% highlight ruby %}
# some code goes here
puts "Hello World!"
{% endhighlight %}

Note that this only provides color-coding. For that you might need to use a front end colorization engine like Highlight.JS or something similar.

MathJax:



## References
Stock, J. H. and Watson, M. W. (1999). "Forecasting inflation," Journal of Monetary Economics,  44(2):293-335.