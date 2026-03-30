+++
date = '2026-02-17T20:43:19+08:00'
title = 'Milkvduo256 Python Lib'
+++


添加文件结构
- buildroot-2021.05
  - package 
    - python-numpy
      - `python-numpy.mk`
      - `python-numpy.hash`
      - `Config.in`
  - configs
    - `milkv-duo256m-sd_musl_riscv64_defconfig`
      - BR2_PACKAGE_PYTHON_NUMPY=y