.. AutoKernel documentation master file, created by
   sphinx-quickstart on Wed Mar 11 12:03:10 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to AutoKernel Docs!
===================================


.. toctree::
  :maxdepth: 1
  :caption: 简介
  :name: sec-introduction

  introduction/introduction

.. toctree::
  :maxdepth: 1
  :caption: 安装
  :name: sec-installation

  installation/install_from_source
  installation/install_from_docker


.. toctree::
  :maxdepth: 2
  :caption: 教程
  :name: sec-tutorial

  tutorials/autosearch
  tutorials/autokernel_plugin
  tutorials/tengine
  tutorials/halide/index

.. toctree::
  :maxdepth: 2
  :caption: 示例
  :name: sec-demo

  demo/gemm_optimization_x86


.. toctree::
  :maxdepth: 1
  :caption: 博文Blog

  blog/autokernel_optimize_gemm_over_200_times_faster
  blog/ai_compiler_overview
