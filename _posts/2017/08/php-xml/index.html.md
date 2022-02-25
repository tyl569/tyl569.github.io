---
layout: post
title: php 创建xml的几种方式
date: 2017-08-24
tags: ["PHP"]
---

#### 1、直接创建字符串

    header("content-type:application/xml;");
    $xml = "<?xml version=\"1.0\" encoding=\"utf-8\"?>
            <people>
            <name>Tyler Teng</name>
            <sex>man</sex>
            </people>";
    echo $xml;

<!--more-->

#### 2、使用DOMDocument进行创建

    header("content-type:application/xml;");
    $xml = new DOMDocument('1.0', 'utf-8');
    $root = $xml->createElement('people');
    $name = $xml->createElement('name', 'Tyler Teng');
    $sex = $xml->createElement('sex', 'man');
    $root->appendChild($name);
    $root->appendChild($sex);
    $xml->appendChild($root);

    echo $xml->saveXML();

#### 3、使用XMLWriter进行创建

    header('Content-type:application/xml');
    $xml_writer = new XMLWriter;
    $xml_writer->openMemory();
    $xml_writer->startDocument('1.0', 'utf-8');
    $xml_writer->startElement('people');
    $xml_writer->writeElement('name', 'Tyler Teng');
    $xml_writer->writeElement('sex', 'man');
    $xml_writer->endElement();
    $xml_writer->endDocument();

    echo $xml_writer->outputMemory();