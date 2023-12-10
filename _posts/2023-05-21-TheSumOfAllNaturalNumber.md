---
layout: post
title: "The Sum of all Natural Numbers"
subtitle: "The Riemann Hypothesis"
author: "Ted"
date: 2023-05-21
header-img: "img/post-bg-infinity.jpg"
header-mask: 0.3
mathjax: true
tags:
  - math
---

This is the most reasonable proof I have encountered so far.                          
The result of $1 + 2 + 3 + 4 + 5 + \ldots + \infty$

Proof:

(1)···········$assume\ S_{1} = 1 - 1 + 1 - 1 + 1 - 1 + \ldots$                                              
(2)···········$\Rightarrow 1 - S_{1} = 1 - (1 - 1 + 1 - 1 + 1 - 1 + \ldots)$                                
(3)···········$\Rightarrow 1 - S_{1} = 1 - 1 + 1 - 1 + 1 - 1 + 1 - \ldots$                                  
(4)···········$\therefore 1 - S_{1} = S_{1}$                                                                
(5)···········$\therefore S_{1} = \frac{1}{2}$                                                              
(6)···········$assume\ S_{2} = 1 - 2 + 3 - 4 + 5 - 6 + \ldots$                                              
(7)···········$\Rightarrow 2S_{2} = (1 - 2 + 3 - 4 + 5 - 6 + \ldots) + (1 - 2 + 3 - 4 + 5 - 6 + \ldots)$    
(8)···········$\Rightarrow 2S_{2} = 1 + ( - 2 + 1) + (3 - 2) + ( - 4 + 3) + (5 - 4) + ( - 6 + 5) + \ldots$  
(9)···········$\Rightarrow 2S_{2} = 1 - 1 + 1 - 1 + 1 - 1 + \ldots$                                         
(10)···········$\Rightarrow 2S_{2} = S_{1} = \frac{1}{2}$                                                   
(11)···········$\therefore S_{2} = \frac{1}{4}$                                                             
(12)···········$assume\ S = 1 + 2 + 3 + 4 + 5 + 6 + \ldots$                                                 
(13)···········$\Rightarrow S - S_{2} = (1 + 2 + 3 + 4 + 5 + 6 + \ldots) - (1 - 2 + 3 - 4 + 5 - 6 + \ldots)$
(14)···········$\Rightarrow S - S_{2} = (1 - 1) + (2 + 2) + (3 - 3) + (4 + 4) + (5 - 5) + (6 + 6) + \ldots$ 
(15)···········$\Rightarrow S - S_{2} = 0 + 4 + 0 + 8 + 0 + 12 + \ldots$                                    
(16)···········$\Rightarrow S - S_{2} = 4(1 + 2 + 3 + 4 + 5 + \ldots)$                                      
(17)···········$\Rightarrow S - S_{2} = 4S$                                                                 
(18)···········$\Rightarrow S = - \frac{S_{2}}{3} = - \frac{\frac{1}{4}}{3} = - \frac{1}{12}$               
(19)···········$Therefore,\ the\ sum\ of\ 1 + 2 + 3 + 4 + 5 + 6 + \ldots = - \frac{1}{12}$                  
