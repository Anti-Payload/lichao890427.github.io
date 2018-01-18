---
layout: post
title: 自己动手写全网最强网盘搜索引擎
categories: Java
description: 自己动手写全网最强网盘搜索引擎
keywords: 
---

## 简介
&emsp;&emsp;是否因为搜不到文件而苦恼？是否因为完不成老板交代的非人类任务而感到迷茫？是否因为不是程序猿而感觉悲伤想重新投胎？不用怕，这篇文章解决你的所有困扰。  
&emsp;&emsp;写这个是看到了网上传的网盘搜索引擎功能不错，为了知道怎么实现的，我进行了逆向，发现了他们的秘密，其实就是用搜索引擎的高级搜索方式，使用关键字，类似site:(pan.baidu.com) title:(动力火车)，把这个搜索语句发给百度或者谷歌进行搜索。了解了原理之后便做了个java界面版的更强大的工具，你不要感到我在吹牛，因为我已经做出来了。很多网站也有搜索功能，不过方式都是类似于这个啦，而且他们支持的都不多不全面而且不能让你调整搜索设置。

![](https://raw.githubusercontent.com/lichao890427/lichao890427.github.io/master/_res/old_blog_7.png)

## Coding

```Java
import java.awt.Color;
import java.awt.event.MouseAdapter;
import java.awt.event.MouseEvent;
import java.awt.event.MouseListener;
import javax.swing.ButtonGroup;
import javax.swing.JButton;
import javax.swing.JCheckBox;
import javax.swing.JFrame;
import javax.swing.JLabel;
import javax.swing.JRadioButton;
import javax.swing.JSeparator;
import javax.swing.SwingConstants;
import javax.swing.JTextField;
public class DevineSearch extends JFrame implements MouseListener
{
 /**
  * 
  */
 private static final long serialVersionUID = 1L;
 private final ButtonGroup searchallconfig = new ButtonGroup();
 private JCheckBox[] wangpanarray=null;
 private JCheckBox[] websitearray=null;
 private JCheckBox[] otherarray=null;
 private JCheckBox allthings=null;
 private JCheckBox wangpan=null;
 private JCheckBox wangzhanluntan=null;
 private JCheckBox others=null;
 public static ResultList result=null;
 public static boolean StopSearch=false;
 private JTextField ToSearch;
 private static final int selallhtmsandfiles=0;
 private static final int selpdfs=1;
 private static final int seldocs=2;
 private static final int selxlss=3;
 private static final int selppts=4;
 private static final int selrtfs=5;
 private static final int selallformats=6;
 private static final int seaanywhere=7;
 private static final int seatitle=8;
 private static final int seaurl=9;
 private static int selformat=selallhtmsandfiles;
 private static int seaplace=seatitle;
 private final ButtonGroup searchplace = new ButtonGroup();
 
 public DevineSearch() 
 {
  super("lichao890427的搜索引擎");
  setTitle("lichao890427的搜索引擎");
  getContentPane().setLayout(null);
  
  JCheckBox baidu = new JCheckBox("\u767E\u5EA6\u7F51\u76D8");
  baidu.setForeground(Color.BLACK);
  baidu.setSelected(true);
  baidu.setBounds(138, 43, 100, 23);
  getContentPane().add(baidu);
  
  JCheckBox _115wangpan = new JCheckBox("115\u7F51\u76D8");
  _115wangpan.setSelected(true);
  _115wangpan.setBounds(240, 43, 100, 23);
  getContentPane().add(_115wangpan);
  
  JCheckBox huawei1 = new JCheckBox("\u534E\u4E3A\u7F51\u76D81");
  huawei1.setSelected(true);
  huawei1.setBounds(342, 43, 100, 23);
  getContentPane().add(huawei1);
  
  JCheckBox huawei2 = new JCheckBox("\u534E\u4E3A\u7F51\u76D82");
  huawei2.setSelected(true);
  huawei2.setBounds(444, 43, 100, 23);
  getContentPane().add(huawei2);
  
  JCheckBox jinshankuaipan = new JCheckBox("\u91D1\u5C71\u5FEB\u76D8");
  jinshankuaipan.setSelected(true);
  jinshankuaipan.setBounds(546, 43, 100, 23);
  getContentPane().add(jinshankuaipan);
  
  JCheckBox lianxiang = new JCheckBox("\u8054\u60F3\u7F51\u76D8");
  lianxiang.setSelected(true);
  lianxiang.setBounds(648, 43, 100, 23);
  getContentPane().add(lianxiang);
  
  JCheckBox yimuhe = new JCheckBox("\u4E00\u6728\u79BE\u7F51\u76D8");
  yimuhe.setSelected(true);
  yimuhe.setBounds(138, 68, 100, 23);
  getContentPane().add(yimuhe);
  
  JSeparator separator = new JSeparator();
  separator.setBackground(Color.BLUE);
  separator.setOrientation(SwingConstants.VERTICAL);
  separator.setForeground(Color.BLUE);
  separator.setBounds(130, 0, 2, 400);
  getContentPane().add(separator);
  
  allthings = new JCheckBox("\u7F51\u9875\u548C\u6587\u4EF6");
  allthings.setBounds(6, 6, 118, 23);
  getContentPane().add(allthings);
  
  wangpan = new JCheckBox("\u7F51\u76D8\u8D44\u6E90");
  wangpan.setSelected(true);
  wangpan.setBounds(6, 43, 118, 23);
  getContentPane().add(wangpan);
  
  wangzhanluntan = new JCheckBox("\u7F51\u7AD9\u8BBA\u575B\u8D44\u6E90");
  wangzhanluntan.setBounds(6, 153, 118, 23);
  getContentPane().add(wangzhanluntan);
  
  JSeparator separator_1 = new JSeparator();
  separator_1.setForeground(Color.BLUE);
  separator_1.setBackground(Color.BLUE);
  separator_1.setBounds(0, 35, 764, 2);
  getContentPane().add(separator_1);
  
  JCheckBox namipan = new JCheckBox("\u7EB3\u7C73\u76D8");
  namipan.setSelected(true);
  namipan.setBounds(240, 68, 100, 23);
  getContentPane().add(namipan);
  
  JCheckBox qianjunwanma = new JCheckBox("\u5343\u519B\u4E07\u9A6C\u7F51\u76D8");
  qianjunwanma.setSelected(true);
  qianjunwanma.setBounds(342, 68, 100, 23);
  getContentPane().add(qianjunwanma);
  
  JCheckBox keleyun = new JCheckBox("\u53EF\u4E50\u4E91\u7F51\u76D8");
  keleyun.setSelected(true);
  keleyun.setBounds(444, 68, 100, 23);
  getContentPane().add(keleyun);
  
  JSeparator separator_2 = new JSeparator();
  separator_2.setForeground(Color.BLUE);
  separator_2.setBackground(Color.BLUE);
  separator_2.setBounds(0, 145, 764, 2);
  getContentPane().add(separator_2);
  
  JRadioButton allhtmsandfiles = new JRadioButton("\u6240\u6709\u7F51\u9875\u548C\u6587\u4EF6");
  allhtmsandfiles.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selallhtmsandfiles;
   }
  });
  allhtmsandfiles.setSelected(true);
  searchallconfig.add(allhtmsandfiles);
  allhtmsandfiles.setBounds(138, 6, 120, 23);
  getContentPane().add(allhtmsandfiles);
  
  JRadioButton pdfs = new JRadioButton("pdf");
  pdfs.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selpdfs;
   }
  });
  searchallconfig.add(pdfs);
  pdfs.setBounds(260, 6, 60, 23);
  getContentPane().add(pdfs);
  
  JRadioButton docs = new JRadioButton("doc");
  docs.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=seldocs;
   }
  });
  searchallconfig.add(docs);
  docs.setBounds(322, 6, 60, 23);
  getContentPane().add(docs);
  
  JRadioButton xlss = new JRadioButton("xls");
  xlss.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selxlss;
   }
  });
  searchallconfig.add(xlss);
  xlss.setBounds(384, 6, 60, 23);
  getContentPane().add(xlss);
  
  JRadioButton ppts = new JRadioButton("ppt");
  ppts.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selppts;
   }
  });
  searchallconfig.add(ppts);
  ppts.setBounds(446, 6, 60, 23);
  getContentPane().add(ppts);
  
  JRadioButton rtfs = new JRadioButton("rtf");
  rtfs.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selrtfs;
   }
  });
  searchallconfig.add(rtfs);
  rtfs.setBounds(506, 6, 60, 23);
  getContentPane().add(rtfs);
  
  JRadioButton allformats = new JRadioButton("\u6240\u6709\u683C\u5F0F");
  allformats.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    selformat=selallformats;
   }
  });
  searchallconfig.add(allformats);
  allformats.setBounds(568, 6, 80, 23);
  getContentPane().add(allformats);
  
  JCheckBox chengtong = new JCheckBox("\u57CE\u901A\u7F51\u76D8");
  chengtong.setSelected(true);
  chengtong.setBounds(546, 68, 100, 23);
  getContentPane().add(chengtong);
  
  JCheckBox xunleikuaichuan = new JCheckBox("\u8FC5\u96F7\u5FEB\u4F20");
  xunleikuaichuan.setSelected(true);
  xunleikuaichuan.setBounds(648, 68, 100, 23);
  getContentPane().add(xunleikuaichuan);
  
  JCheckBox _360yunchuan = new JCheckBox("360\u4E91\u4F20");
  _360yunchuan.setSelected(true);
  _360yunchuan.setBounds(138, 91, 100, 23);
  getContentPane().add(_360yunchuan);
  
  JCheckBox weipan1 = new JCheckBox("\u5A01\u76D81");
  weipan1.setSelected(true);
  weipan1.setBounds(240, 93, 100, 23);
  getContentPane().add(weipan1);
  
  JCheckBox rayfile = new JCheckBox("rayfile\u7F51\u76D8");
  rayfile.setSelected(true);
  rayfile.setBounds(444, 91, 100, 23);
  getContentPane().add(rayfile);
  
  JCheckBox xunzai = new JCheckBox("\u8FC5\u8F7D\u7F51\u76D8");
  xunzai.setSelected(true);
  xunzai.setBounds(546, 91, 100, 23);
  getContentPane().add(xunzai);
  
  JCheckBox _163wangpan = new JCheckBox("163\u7F51\u76D8");
  _163wangpan.setSelected(true);
  _163wangpan.setBounds(648, 93, 100, 23);
  getContentPane().add(_163wangpan);
  
  JCheckBox verycd = new JCheckBox("verycd\u79CD\u5B50");
  verycd.setBounds(138, 153, 100, 23);
  getContentPane().add(verycd);
  
  JCheckBox ed2000 = new JCheckBox("ed2000\u79CD\u5B50");
  ed2000.setBounds(240, 153, 100, 23);
  getContentPane().add(ed2000);
  
  JCheckBox xinlangaiwen = new JCheckBox("\u7231\u95EE\u5171\u4EAB");
  xinlangaiwen.setBounds(444, 153, 100, 23);
  getContentPane().add(xinlangaiwen);
  
  JCheckBox weipan2 = new JCheckBox("\u5A01\u76D82");
  weipan2.setSelected(true);
  weipan2.setBounds(342, 93, 100, 23);
  getContentPane().add(weipan2);
  
  JCheckBox qianyi = new JCheckBox("\u5343\u6613\u7F51\u76D8");
  qianyi.setSelected(true);
  qianyi.setBounds(138, 116, 100, 23);
  getContentPane().add(qianyi);
  
  others = new JCheckBox("\u5F71\u89C6|\u8D44\u6599|\u5176\u4ED6");
  others.setBounds(6, 240, 118, 23);
  getContentPane().add(others);
  
  JCheckBox bthome = new JCheckBox("bt\u4E4B\u5BB6");
  bthome.setBounds(342, 153, 100, 23);
  getContentPane().add(bthome);
  
  JCheckBox dajialuntan = new JCheckBox("\u5927\u5BB6\u8BBA\u575B");
  dajialuntan.setBounds(546, 153, 100, 23);
  getContentPane().add(dajialuntan);
  
  JCheckBox qiannao = new JCheckBox("\u5343\u8111\u4E0B\u8F7D");
  qiannao.setBounds(648, 153, 100, 23);
  getContentPane().add(qiannao);
  
  JCheckBox _51cto = new JCheckBox("51cto\u4E0B\u8F7D");
  _51cto.setBounds(138, 178, 100, 23);
  getContentPane().add(_51cto);
  
  JCheckBox csdn = new JCheckBox("CSDN\u4E0B\u8F7D");
  csdn.setBounds(240, 178, 100, 23);
  getContentPane().add(csdn);
  
  JCheckBox xixi = new JCheckBox("\u897F\u897F\u4E0B\u8F7D");
  xixi.setBounds(342, 178, 100, 23);
  getContentPane().add(xixi);
  
  JSeparator separator_3 = new JSeparator();
  separator_3.setForeground(Color.BLUE);
  separator_3.setBackground(Color.BLUE);
  separator_3.setBounds(0, 232, 764, 2);
  getContentPane().add(separator_3);
  
  JCheckBox baiduwenku = new JCheckBox("\u767E\u5EA6\u6587\u5E93");
  baiduwenku.setBounds(240, 240, 100, 23);
  getContentPane().add(baiduwenku);
  
  JCheckBox xuexiziliaoku = new JCheckBox("\u5B66\u4E60\u8D44\u6599\u5E93");
  xuexiziliaoku.setBounds(342, 240, 100, 23);
  getContentPane().add(xuexiziliaoku);
  
  JCheckBox lanying = new JCheckBox("\u84DD\u5F71\u8BBA\u575B");
  lanying.setBounds(240, 290, 100, 23);
  getContentPane().add(lanying);
  
  JCheckBox zhenhao = new JCheckBox("\u771F\u597D\u8BBA\u575B");
  zhenhao.setBounds(342, 290, 100, 23);
  getContentPane().add(zhenhao);
  
  JCheckBox googlecode = new JCheckBox("googlecode");
  googlecode.setBounds(240, 340, 100, 23);
  getContentPane().add(googlecode);
  
  JCheckBox pudn = new JCheckBox("pudn");
  pudn.setBounds(342, 340, 100, 23);
  getContentPane().add(pudn);
  
  JCheckBox jiaobenzhijia = new JCheckBox("\u811A\u672C\u4E4B\u5BB6");
  jiaobenzhijia.setBounds(444, 240, 100, 23);
  getContentPane().add(jiaobenzhijia);
  
  JLabel label_1 = new JLabel("\u8D44\u6599\uFF1A");
  label_1.setBounds(142, 242, 44, 19);
  getContentPane().add(label_1);
  
  JLabel label_2 = new JLabel("\u7535\u5F71\uFF1A");
  label_2.setBounds(142, 292, 44, 19);
  getContentPane().add(label_2);
  
  JLabel label_3 = new JLabel("\u4EE3\u7801\uFF1A");
  label_3.setBounds(142, 342, 44, 19);
  getContentPane().add(label_3);
  
  JSeparator separator_4 = new JSeparator();
  separator_4.setForeground(Color.BLUE);
  separator_4.setBackground(Color.BLUE);
  separator_4.setBounds(0, 398, 764, 2);
  getContentPane().add(separator_4);
  
  JButton search = new JButton("\u5F00\u59CB\u641C\u7D22");
  search.setBounds(648, 410, 116, 23);
  getContentPane().add(search);
  
  ToSearch = new JTextField();
  ToSearch.setBounds(6, 411, 640, 21);
  getContentPane().add(ToSearch);
  ToSearch.setColumns(10);
  
  JLabel lbllichao = new JLabel("\u4F5C\u8005\u4FE1\u606F:    lichao890427    qq:571652571    \u7FA4:124408915    lichao.890427@163.com");
  lbllichao.setForeground(Color.RED);
  lbllichao.setHorizontalAlignment(SwingConstants.LEFT);
  lbllichao.setBounds(6, 473, 560, 19);
  getContentPane().add(lbllichao);
  
  JLabel label = new JLabel("\u641C\u7D22\u5173\u952E\u8BCD\u4F4D\u4E8E\uFF1A");
  label.setBounds(6, 448, 100, 15);
  getContentPane().add(label);
  
  JRadioButton searchanywhere = new JRadioButton("\u7F51\u9875\u7684\u4EFB\u4F55\u5730\u65B9");
  searchanywhere.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    seaplace=seaanywhere;
   }
  });
  searchplace.add(searchanywhere);
  searchanywhere.setBounds(130, 444, 128, 23);
  getContentPane().add(searchanywhere);
  
  JRadioButton searchtitle = new JRadioButton("\u4EC5\u7F51\u9875\u7684\u6807\u9898\u4E2D");
  searchtitle.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    seaplace=seatitle;
   }
  });
  searchplace.add(searchtitle);
  searchtitle.setSelected(true);
  searchtitle.setBounds(261, 444, 121, 23);
  getContentPane().add(searchtitle);
  
  JRadioButton searchurl = new JRadioButton("\u4EC5\u7F51\u9875\u7684URL\u4E2D");
  searchurl.addMouseListener(new MouseAdapter() 
  {
   @Override
   public void mouseClicked(MouseEvent arg0) 
   {
    seaplace=seaurl;
   }
  });
  searchplace.add(searchurl);
  searchurl.setBounds(385, 444, 121, 23);
  getContentPane().add(searchurl);
  
  try
  {
   wangpanarray=new JCheckBox[]
   {
    baidu,   _115wangpan, huawei1,  huawei2,  jinshankuaipan, lianxiang,
    yimuhe,   namipan,  qianjunwanma, keleyun,  chengtong,  xunleikuaichuan,
    _360yunchuan, weipan1,  weipan2,  rayfile,  xunzai,   _163wangpan,
    qianyi,
   };
   
   websitearray=new JCheckBox[]
   {
    verycd,   ed2000,   bthome,   xinlangaiwen, dajialuntan, qiannao,
    _51cto,   csdn,   xixi,
   };
   
   otherarray=new JCheckBox[]
   {
    baiduwenku,  xuexiziliaoku, jiaobenzhijia, 
    lanying,  zhenhao,
    googlecode,  pudn,
   };
   
   for(int i=0;i<wangpanarray.length;i++)
   {
    wangpanarray[i].addMouseListener(new MouseAdapter()
    {
     @Override
     public void mouseClicked(MouseEvent arg0) 
     {
      allthings.setSelected(false);
      others.setSelected(false);
     }
    });
   }
   
   for(int i=0;i<websitearray.length;i++)
   {
    websitearray[i].addMouseListener(new MouseAdapter()
    {
     @Override
     public void mouseClicked(MouseEvent arg0) 
     {
      allthings.setSelected(false);
      others.setSelected(false);
     }  
    });
   }
   
   for(int i=0;i<otherarray.length;i++)
   {
    otherarray[i].addMouseListener(new MouseAdapter()
    {
     @Override
     public void mouseClicked(MouseEvent arg0) 
     {
      allthings.setSelected(false);
      others.setSelected(false);
     } 
    });
   }
   
   allthings.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     if(allthings.isSelected())
     {
      wangpan.setSelected(false);
      for(int i=0;i<wangpanarray.length;i++)
      {
       wangpanarray[i].setSelected(false);
      }
      wangzhanluntan.setSelected(false);
      for(int i=0;i<websitearray.length;i++)
      {
       websitearray[i].setSelected(false);
      } 
      others.setSelected(false);
      for(int i=0;i<otherarray.length;i++)
      {
       otherarray[i].setSelected(false);
      }
     }
    }
   });
   
   wangpan.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     allthings.setSelected(false);
     if(wangpan.isSelected())
     {
      for(int i=0;i<wangpanarray.length;i++)
      {
       wangpanarray[i].setSelected(true);
      }
     }
     else
     {
      for(int i=0;i<wangpanarray.length;i++)
      {
       wangpanarray[i].setSelected(false);
      }
     }
    }
   });
   
   wangzhanluntan.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     allthings.setSelected(false);
     if(wangzhanluntan.isSelected())
     {
      for(int i=0;i<websitearray.length;i++)
      {
       websitearray[i].setSelected(true);
      }
     }
     else
     {
      for(int i=0;i<websitearray.length;i++)
      {
       websitearray[i].setSelected(false);
      }
     }
    }
   });
   
   others.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     allthings.setSelected(false);
     if(others.isSelected())
     {
      for(int i=0;i<otherarray.length;i++)
      {
       otherarray[i].setSelected(true);
      }
     }
     else
     {
      for(int i=0;i<otherarray.length;i++)
      {
       otherarray[i].setSelected(false);
      }
     }
    }
   });
   
   search.addMouseListener(this);
  }
  catch(Exception e)
  {
   e.printStackTrace();
  }
  
  setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
  setSize(784,547);
  setVisible(true);
 }
 
 public static void main(String[] args)
 {
  try
  {
   DevineSearch cursearch=new DevineSearch();
   DevineSearch.result=new ResultList(cursearch);
  }
  catch(Exception e)
  {
   e.printStackTrace();
  }
 }
 @Override
 public void mouseClicked(MouseEvent arg0) 
 {
  StopSearch=false;
  result.ClearDefaultTable();
  
  try
  {
   String searchstring="http://www.baidu.com/s?q1="+ToSearch.getText()+"&q2=&q3=&q4=&rn=50&lm=0&ct=0&ft=";
   String seawhstring="&q5=1&q6=";
   if(seaplace == seaanywhere)
    seawhstring="&q5=&q6=";
   else if(seaplace == seaurl)
    seawhstring="&q5=2&q6=";
   
   if(allthings.isSelected())
   {
    switch(selformat)
    {
     case selallhtmsandfiles:
      break;
      
     case selpdfs:
      searchstring+="pdf";
      break;
      
     case seldocs:
      searchstring+="doc";
      break;
      
     case selxlss:
      searchstring+="xls";
      break;
      
     case selppts:
      searchstring+="ppt";
      break;
      
     case selrtfs:
      searchstring+="rtf";
      break;
      
     case selallformats:
      searchstring+="all";
      break;
      
     default:
      break;
    }
    result.SearchOne(searchstring+seawhstring,true);
   }
   else if(wangpan.isSelected() || wangzhanluntan.isSelected() || others.isSelected())
   {
    String wangpanarraystr[]=
    {
     "pan.baidu.com", "q.115.com",  "dl.dbank.com",  "dl.vmall.com",  "www.kuaipan.cn", "app.lenovo.com",
     "www.yimuhe.com", "www.namipan.cc", "7958.com",   "www.colafile.com", "www.400gb.com", "kuai.xunlei.com",
     "yunpan.cn",  "vdisk.cn",   "vdisk.weibo.com", "www.rayfile.com", "u.xunzai.com",  "www.163disk.com",
     "1000eb.com",
    };
    
    String websitearraystr[]=
    {
     "www.verycd.com", "www.ed2000.com",  "www.btbbt.com", "ishare.iask.sina.com.cn", "club.topsage.com", "www.qiannao.com",
     "down.51cto.com", "download.csdn.net", "www.cr173.com",
    };
    
    String otherarraystr[]=
    {
     "wenku.baidu.com", "www.xuexi111.com",  "www.jb51.net",
     "www.blue08.cn", "www.chinazhw.com",
     "googlecode.com", "www.pudn.com",
    };
   
    searchstring+=seawhstring;
    for(int i=0;i<wangpanarray.length && !StopSearch;i++)
    {
     if(wangpanarray[i].isSelected())
     {
      result.SearchOne(searchstring+wangpanarraystr[i],true);
     }
    }
    for(int i=0;i<websitearray.length && !StopSearch;i++)
    {
     if(websitearray[i].isSelected())
     {
      result.SearchOne(searchstring+websitearraystr[i],true);
     }
    } 
    for(int i=0;i<otherarray.length && !StopSearch;i++)
    {
     if(otherarray[i].isSelected())
     {
      result.SearchOne(searchstring+otherarraystr[i],true);
     }
    }
   } 
  }
  catch(Exception e)
  {
   e.printStackTrace();
  }
 }
 @Override
 public void mouseEntered(MouseEvent arg0){}
 @Override
 public void mouseExited(MouseEvent arg0){}
 @Override
 public void mousePressed(MouseEvent arg0){}
 @Override
 public void mouseReleased(MouseEvent arg0){}
}


import java.awt.Rectangle;
import java.awt.Toolkit;
import java.awt.datatransfer.StringSelection;
import java.awt.event.ComponentAdapter;
import java.awt.event.ComponentEvent;
import java.awt.event.MouseAdapter;
import java.awt.event.MouseEvent;
import java.awt.event.WindowAdapter;
import java.awt.event.WindowEvent;
import java.awt.event.WindowStateListener;
import java.util.LinkedList;
import java.util.Queue;
import javax.swing.JButton;
import javax.swing.JFrame;
import javax.swing.JScrollPane;
import javax.swing.JTable;
import javax.swing.table.DefaultTableModel;
import javax.swing.JLabel;
import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.select.Elements;
public class ResultList extends JFrame
{
 /**
  * 
  */
 private static final long serialVersionUID = -7586074484755866311L;
 public JTable table=null;
 private JScrollPane sp=null;
 private JButton copyaddress=null;
 private JButton stop=null;
 private int index=0;
 private Queue<MySearchThread> threadqueue=null;
 private JLabel state=null;
 private String errorstring=null;
 
 class MySearchThread extends Thread
 {
  private String cururl=null;
  private boolean findsub=false;
  public MySearchThread(String cururl,boolean findsub)
  {
   super();
   this.cururl=cururl;
   this.findsub=findsub;
   System.out.println(cururl);
  }
  
  @Override
  public void run()
  {
   errorstring=null;
   try
   {
    if(DevineSearch.StopSearch)
    {
     return;
    }
    
    Document doc=Jsoup.connect(cururl).timeout(0).get();//源1  
    Elements curresult=doc.select("table.result");
       for(Element e:curresult)
       {
        String[] arr=new String[]{""+index,"","",""};
        Elements rt1=e.select(".t a");
        if(!rt1.isEmpty())
        {
         arr[1]=RemoveEm(rt1.get(0).text());
         arr[3]=RemoveEm(rt1.get(0).attr("href"));
        }
        rt1=e.select(".c-abstract");
        if(!rt1.isEmpty())
        {
         arr[2]=RemoveEm(rt1.get(0).text());
        }
        DefaultTableModel tableModel=(DefaultTableModel)table.getModel();
        tableModel.addRow(arr);
        index++;
       }  
       
       if(findsub)
       {
     Elements otherresults=doc.select("p#page a");
     for(Element other:otherresults)
     {
      if(!other.text().contains("下一页"));
      {
       threadqueue.add(new MySearchThread(other.attr("abs:href"),false));
      }
     }
       }
   }
   catch(Exception e)
   {
    e.printStackTrace();
    errorstring=e.getCause().toString();
   } 
  }
 }
 
 public void ClearDefaultTable()
 {
  index=0;
  if(table == null)
   return;
  table.setModel(new DefaultTableModel(
    new Object[][] {
    },
    new String[] {
     "\u5E8F\u53F7", "\u6807\u9898", "\u5176\u4ED6\u4FE1\u606F", "\u7F51\u5740"
    }
   ));
 }
 
 public ResultList(DevineSearch father) 
 {
  super("搜索结果");
  setDefaultCloseOperation(JFrame.DO_NOTHING_ON_CLOSE);
  
  addWindowListener(new WindowAdapter()
  {
   @Override
   public void windowClosing(WindowEvent arg0) 
   {
    DevineSearch.result=null;
    DevineSearch.StopSearch=true;
   } 
  });
  addWindowStateListener(new WindowStateListener() 
  {
   @Override
   public void windowStateChanged(WindowEvent e)
   {  
    Rectangle rtb=getContentPane().getBounds();
    if(sp != null)
    {
     Rectangle rta=sp.getBounds();
     sp.setBounds(rta.x,rta.y,rtb.width,rtb.height-30);
     table.setBounds(rta.x,rta.y,rtb.width,rtb.height-30);
    }
    if(copyaddress != null)
    {
     Rectangle rta=copyaddress.getBounds();
     copyaddress.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height);
    }
    if(stop != null)
    {
     Rectangle rta=stop.getBounds();
     stop.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height);
    }
    if(state != null)
    {
     Rectangle rta=state.getBounds();
     state.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height); 
    }
   }
  });
  addComponentListener(new ComponentAdapter() 
  {
   @Override
   public void componentResized(ComponentEvent arg0) 
   {
    Rectangle rtb=getContentPane().getBounds();
    if(sp != null)
    {
     Rectangle rta=sp.getBounds();
     sp.setBounds(rta.x,rta.y,rtb.width,rtb.height-30);
    }
    if(copyaddress != null)
    {
     Rectangle rta=copyaddress.getBounds();
     copyaddress.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height);
    }
    if(stop != null)
    {
     Rectangle rta=stop.getBounds();
     stop.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height);
    }
    if(state != null)
    {
     Rectangle rta=state.getBounds();
     state.setBounds(rta.x,rtb.y+rtb.height-rta.height,rta.width,rta.height);    
    }
   }
  });
  
  setTitle("搜索结果");
  getContentPane().setLayout(null);
  
  stop = new JButton("\u505C\u6B62\u641C\u7D22");
  stop.setBounds(32, 339, 93, 23);
  getContentPane().add(stop);
  
  copyaddress = new JButton("\u590D\u5236\u94FE\u63A5");
  copyaddress.setBounds(150, 339, 93, 23);
  getContentPane().add(copyaddress);
  
  table = new JTable();
  ClearDefaultTable();
  table.getColumnModel().getColumn(0).setPreferredWidth(50);
  table.getColumnModel().getColumn(0).setMinWidth(50);
  table.getColumnModel().getColumn(1).setPreferredWidth(250);
  table.getColumnModel().getColumn(2).setPreferredWidth(200);
  table.getColumnModel().getColumn(3).setPreferredWidth(200);
  table.setBounds(10, 10, 564, 288);
  sp=new JScrollPane(table);
  sp.setBounds(0, 0, 584, 329);
  getContentPane().add(sp);
  
  state = new JLabel("");
  state.setBounds(270, 339, 271, 19);
  getContentPane().add(state);
    
  try
  {
   stop.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     DevineSearch.StopSearch=true;
    }
   });
   
   copyaddress.addMouseListener(new MouseAdapter() 
   {
    @Override
    public void mouseClicked(MouseEvent arg0) 
    {
     if(table.getSelectedRowCount() > 0)
     {
      String cursel=(String)table.getValueAt(table.getSelectedRow(),3);
      Toolkit.getDefaultToolkit().getSystemClipboard().setContents(new StringSelection(cursel),null);    
     }
    }
   });
   
  }
  catch(Exception e)
  {
   e.printStackTrace();
  }
  setSize(600,400);
  setVisible(true);
  
  threadqueue=new LinkedList<MySearchThread>();
  new Thread()
  {
   @Override
   public void run()
   {
    try
    {
     while(true)
     {
      Thread.sleep(100);
      if(!threadqueue.isEmpty())
      {
       state.setText("处理中");
       MySearchThread cur=threadqueue.poll();
       cur.start();
       cur.join();
       if(errorstring == null)
        state.setText("处理完毕");
       else
        state.setText("出错："+errorstring);
      }
      else
       state.setText("空闲");
     }
    }
    catch(Exception e)
    {
     e.printStackTrace();
    } 
   }
  }.start();
 }
 
 public void SearchOne(String url,boolean findsub)
 {
  threadqueue.add(new MySearchThread(url,true));
 }
 
 private String RemoveEm(String src)
 {
  src.replace("<em>","");
  src.replace("</em>","");
  return src;
 }
 
 public static void main(String[] args)
 {
  new ResultList(null).SearchOne("",true);
 }
}

```