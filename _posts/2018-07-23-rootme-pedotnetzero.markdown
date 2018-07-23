---
layout: post
title: Root-me) PE .NET - 0 Protection
date: 2018-07-23 10:32:20 +0900
description: # Add post description (optional)
img: # Add image post (optional)
tags: [CTF, Root-me]
author: purelledhand # Add name author (optional)
---
C# 디컴파일러를 이용해서 보니까 valider 버튼 클릭 이벤트 함수에서 바로 input과 password를 비교하는 것을 확인할 수 있었다.

    private void Button1_Click(object sender, EventArgs e)
    {
        if (Operators.CompareString(this.TextBox1.Text, "DotNetOP", false) == 0)
        {
            int num1 = (int) Interaction.MsgBox((object) "Bravo! Vous pouvez valider avec ce mot de passe\r\nWell done! You can validate with this password)
        }
        else
        {
            int num2 = (int) Interaction.MsgBox((object) "Mauvais mot de passe\r\nBad password", MsgBoxStyle.OkOnly, (object) null);
        }
    }


<img src="{{site.baseurl}}/assets/img/rootme/cracking/pedotnetzero.PNG" alt="flow" style="width: 500px; margin: 0 auto;"/>