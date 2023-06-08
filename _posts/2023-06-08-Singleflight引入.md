---
layout:     post
title:      Singleflight调研和引入
subtitle:   记录Singleflight
date:       2023-06-08
author:     zhiqing
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - golang package
---
<h2>Singleflight调研和引入</h2>
记录一下最近在项目中遇到的一个优化点的一些想法:Singleflight

<p><strong>概念</strong></p>

<p><code>singleflight</code> 包是 Go 标准库中的一个用于减少重复函数调用的机制。它提供了一种可以合并相同函数调用的方法，避免了不必要的重复计算和请求。</p>

<p>在并发环境下，有时会出现多个 goroutine 同时发起相同的请求的情况，这可能会导致不必要的资源浪费和性能瓶颈。 <code>singleflight</code> 可以解决这个问题，使得只会有一个 goroutine 真正地进行请求和计算，其他 goroutine 则直接等待其结果，从而提高效率并减少不必要的计算。</p>

<p>使用 <code>singleflight</code> 的方式很简单，主要包含以下两个步骤：</p>

<ol>
  <li>定义一个 <code>Group</code> 对象。一个 <code>Group</code> 对象代表一个函数的集合，每个函数都可以被调用多次。</li>
  <li>使用 <code>Do</code> 方法来执行函数调用，并传入要调用的函数名和参数。当多个 goroutine 调用相同的函数时，只有第一个 goroutine 会真正地调用该函数，其他 goroutine 则直接等待其结果，并返回该结果。</li>
</ol>

<p><strong>示例代码</strong></p>

<p>这里我写了一个简单的测试来展示singleflight的效果</p>

<pre><code><span class="hljs-keyword">import</span> (
   <span class="hljs-string">"golang.org/x/sync/singleflight"</span>
   <span class="hljs-string">"testing"</span>)
   
<span class="hljs-keyword">func</span> init() {
   InitDB()
}

<span class="hljs-keyword">var</span> group singleflight.Group

<span class="hljs-comment">//随便定义的数据库查询操作</span>
<span class="hljs-keyword">func</span> op() (interface{}, error) {
   <span class="hljs-keyword">var</span> total int64
   DB.Model(&amp;UserLogin{}).Count(&amp;total)
   <span class="hljs-keyword">return</span> nil, nil
}

<span class="hljs-comment">//使用singleflight的并发测试</span>
<span class="hljs-keyword">func</span> BenchmarkParallelSingleFlightOP(b *testing.B) {
   b.SetParallelism(32)  <span class="hljs-comment">//启动32个goroutine进行并发测试</span>
   b.RunParallel(<span class="hljs-function"><span class="hljs-keyword">func</span>(pb *testing.PB) {
      <span class="hljs-keyword">for</span> pb.Next() {
         group.Do(<span class="hljs-string">"key"</span>, op) <span class="hljs-comment">// 这里所有请求都相同所以将key写死了，实际应用中可以不同</span>
      }
   </span>})
}

<span class="hljs-comment">//不使用singleflight的并发测试</span>
<span class="hljs-keyword">func</span> BenchmarkParallelOP(b *testing.B) {
   b.SetParallelism(32)  <span class="hljs-comment">//启动32个goroutine进行并发测试</span>
   b.RunParallel(<span class="hljs-function"><span class="hljs-keyword">func</span>(pb *testing.PB) {
      <span class="hljs-keyword">for</span> pb.Next() {
         op()
      }
   </span>})
}

</code></pre>

<p>在这个示例代码中，<code>group</code> 是一个 <code>singleflight.Group</code> 对象，<code>Do</code> 方法是该对象提供的方法。 <code>Do</code> 方法接受两个参数：第一个参数是一个字符串类型的 key，用于标识要执行的函数；第二个参数是一个函数类型，该函数会在第一次调用时执行并返回一个结果。</p>

<p>如果同时有多个 goroutine 调用了 <code>Do</code> 方法，并且它们的 <code>key</code> 参数相同，则只有第一个 goroutine 会真正地执行该函数，其他 goroutine 则直接等待并返回其结果。注意，结果可能是一个错误值，因此需要检查错误并处理。</p>

<p>需要注意的是，由于 <code>Do</code> 方法自身是带有锁机制的，因此在处理高并发场景时，应该注意不要在 <code>Do</code> 方法中执行过多的耗时操作，以避免阻塞其他请求和影响程序性能。</p>

<p><strong>测试结果</strong></p>

<p>使用 <code>singleflight</code> 时</p>

<pre><code>goos: linux
goarch: amd64
pkg: demo/models
cpu: AMD Ryzen 7 6800H with Radeon Graphics
BenchmarkParallelSingleFlightOP
BenchmarkParallelSingleFlightOP-16         70171             16227 ns/op
PASS
</code></pre>

<p>未使用 <code>singleflight</code> 时:</p>
<img src="img/singleflight.png">
<p>MySQL 因为并发度过大导致每条查询语句速度都很慢，可见这是一个不错的优化方案。</p>

