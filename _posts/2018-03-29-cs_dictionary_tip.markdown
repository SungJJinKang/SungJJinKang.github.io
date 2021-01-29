---
layout: post
title:  "구글 플레이 스토어의 앱 버전을 서버없이 검사하는 방법"
date:   2018-12-09
categories: Unity
---

요구조건 :    
앱 버전을 x.x.x 형식으로 한다.

- 업데이트를 강제 하고 싶지 않은 경우 맨 뒤의 숫자만 바꾸면 업데이트 요구를 하지 않는다

```c#
bool IsSameVersion = false; //일단은 그냥 같은 버젼으로 취급해준다
bool IsChecked = false;
      
private void GooglePlayVersionCheck()
{
        UnsafeSecurityPolicy.Instate(); //이 코드를 안쓸 경우 오류가 뜨더라구요

        string marketVersion = "";

        string url = "https://play.google.com/store/apps/details?id=앱id";

        HtmlWeb web = new HtmlWeb();
        HtmlDocument doc = web.Load(url);
        yield return doc;

        try
        {
            System.Collections.Generic.IEnumerable<HtmlNode> nodes = doc.DocumentNode.Descendants("div").Where(d => d.Attributes["class"].Value.Contains("htlgb"));

            foreach (HtmlNode node in doc.DocumentNode.SelectNodes("//span[@class='htlgb']"))
            {
                string value = node.InnerText.Trim();

                /////////
                if (System.Text.RegularExpressions.Regex.IsMatch(value, @"^\d{1}\.\d{1}\.\d{1}$"))
                {  
                    marketVersion = value;
                    Debug.Log("market version : " + marketVersion.Trim());

                    if(marketVersion.Split('.')[0] == Application.version.Split('.')[0] && marketVersion.Split('.')[1] == Application.version.Split('.')[1]) // . . 형식중에서 앞의 두 숫자가 바뀌는 경우에만 업데이트 요구 -> 업데이트를 강제할 필요없는 경우에는 맨뒤의 숫자만 바꾸자
                    {
                        Debug.Log("업데이트 필요없음");
                        //버전이 같은 경우 
                        IsSameVersion = true; //일단은 그냥 같은 버젼으로 취급해준다
                        IsChecked = true;
                        yield break; //함수 빠져나감 
                    }
                    else
                    {
                        Debug.Log("업데이트 필요함");
                        //버전이 다른 경우 
                        IsSameVersion = false;
                        IsChecked = true;
                        _DeveloperLogoScene.SetActiveRequestUpdateGameOnGooglePlayPanel(true);
                        yield break; //함수 빠져나감 
                    }
                }
            }
        }
        catch
        {
            //구글 플레이 쪽에서 코드를 자주 바꾸다 버전을 못가져 올 수 도 있습니다. 이런 경우를 대비하여 exception이 뜬 경우 그냥 버전이 동일한걸로 취급
            IsSameVersion = true; //일단은 그냥 같은 버젼으로 취급해준다
            IsChecked = true;
        }
  }
```

