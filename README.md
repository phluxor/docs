# Phluxor Documentation

A toolkit for flexible actor models in PHP, empowering the PHP ecosystem.

Requires PHP 8.3 and swoole 5.0 or later.

Welcome to Phluxor, a open-source toolkit for building actor models in PHP.  
It is designed to be flexible and easy to use, and it is built on top of the Swoole extension.  

Phluxorは、PHPでアクターモデルを構築するためのオープンソースツールキットです。  
アクターモデルを柔軟で使いやすい設計になっており、Swoole拡張機能をベースに構築しています。  

[Getting Started Guide](en/guide/index.md)  
[日本語ガイド](ja/guide/index.md)  

## Simpler Concurrent

By using actors based on PHP and Swoole, you can build systems that efficiently use server resources and scale out across multiple servers.

Leveraging the PHP ecosystem while using the actor model allows for the creation of flexible systems that differ from traditional PHP applications.

PHPとSwooleをベースとしたアクターを利用することで、サーバーリソースを効率的に使用したり、  
複数のサーバーを使用してスケールアウトできるシステムを構築できます。

PHPのエコシステムを活用しながら、アクターモデルを利用することで、  
従来のPHPアプリケーションとは異なる柔軟なシステムを構築できます。

## Resilient

Phluxor supports system development based on the principles of the reactive manifesto. Unlike traditional PHP applications,  
it enables systems that can self-heal in case of failures while maintaining responsiveness.

Phluxorを利用することでリアクティブ宣言の原則に基づいたシステム開発がサポートされます。  
これまでのPHPアプリケーションとことなり障害発生時の自己修復や、応答性を維持するシステムを実現できます。

## Elastic & Decentralized

Phluxor is designed based on the actor model, with all actors operating independently. 　
This makes it easier to support load balancing and scalability.

It facilitates the development of advanced systems with load balancing and routing, 　
as well as distributed systems without a single point of failure.

Phluxor also supports Event Sourcing and CQRS, enhancing system flexibility by separating data persistence from data reading and writing.

Phluxorはアクターモデルをベースに設計されており、すべてのアクターは独立して動作します。  
このため負荷分散やスケーラビリティに対応しやすい特徴があります。  
単一障害点のない分散システムや、負荷分散やルーティングで高度なシステム開発をサポートします
Event Sourcing and CQRSもサポートされており、データの永続化やデータの読み書きを分離することで、
システムの柔軟性を高めることができます。
