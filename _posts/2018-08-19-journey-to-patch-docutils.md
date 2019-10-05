---
layout: post
title: "Journey to patch docutils"
description: "Nekoyume to sphinx, sphinx to docutils"

tags: [opensource, contribution, nekoyume, sphinx, docutils]
---

[PYCON KOREA 2018](https://www.pycon.kr/2018/)의 컨퍼런스 일정을 구경하던 중, [nekoyume](https://nekoyu.me)라는 프로젝트를 발견했다.
블록체인 + RPG라니, 그냥 지나치기 어려운 조합이지 않은가!

 플레이 컨셉과 프로젝트 구현을 위해 사용한 기술 등 정보를 얻기 위해 [nekoyume 백서](https://docs.nekoyu.me/white_paper_ko.html)를 읽기 시작했다.

# **Missing herf links**
백서를 훑어보던 중, 사이드바의 링크가 제대로 동작하지 않는 이슈가 발생했다. 원인을 확인하기 위해 HTML 소스를 훑어봤는데, `href="#"`와 같이 *internal link*가 제대로 링크되어있지 않음을 확인할 수 있었다. 

왜 이슈가 발생하는지 root cause를 찾기 위해 더 훑어본 결과, 몇 가지 규칙을 찾을 수 있었다.

<a id="rules"></a>
1. 모든 heading tag는 wrapping div가 있다.
2. wrapping **div의 id는 heading tag 내부의 text와 동일**하다. <br>(단, text 내의 공백은 `-`(dash)로 대체된다.)
3. **(아마도) ascii character, 그중에서도 `[a-z0-9]`, 만 id로 등록된다.** <br>즉, **non-ascii character는 모두 drop**된다.
4. 사이드바의 *internal link*는 wrapping div의 id를 이용한다.

 heading tag 내부의 text에 `[a-z0-9]`가 한 글자도 없을 시 `<div class="section" id="">`와 같이 id가 >>빈 문자열<<이였고, 이 경우 `href="#"`와 같이 *internal link*가 제대로 링크되지 않은 것이었다.
  
# **Solve the issue**
 위의 규칙을 찾고 나서, 한 가지 아이디어를 떠올릴 수 있었다.
 
 2번 규칙을 다시 말하면, heading tag 내부에 `[a-z0-9]`를 만족하는 글자가 하나라도 존재할 시 id가 생성된다고 말할 수 있다. 즉, **heading tag 내부에 조건을 만족하는 ascii character를 포함하는 comment를 작성**하면 위 이슈를 해결할 수 있다!
 
 단, `.md`, `.html` 두 파일 포맷에서 comment 형식을 만족시켜야 하기 때문에 `<!-- -->` 스타일을 이용해야 한다.

 위 아이디어를 이용해 [PR](https://github.com/nekoyume/nekoyume/pull/92)을 보냈고, nekoyume:master에 merge되어 본 이슈는 해결된 상태이다. (아직 nekoyume:gh-pages가 업데이트되지 않은 것으로 보아 merge이후 deploy는 진행하지 않은 것 같다.)
 
# **Journey to resolve the issue**
 하지만 어떻게 이슈를 근본적으로 해결할 수 있을지 고민해보기 위해서, 백서가 빌드되는 과정을 하나하나 쫓아가며 모든 과정을 의심해보기로 했다.

## environment
 백서의 내용은 markdown을 통해 작성되었고, [sphinx](https://github.com/sphinx-doc/sphinx)를 통해 자동으로 빌드하고 있었다. markdown-based였기 때문에 sphinx 내부 빌드 과정에서 [recommonmark](https://recommonmark.readthedocs.io/en/latest/index.html)의 CommonMarkParser를 이용해 markdown을 parsing하고 있었다.
 
## [recommonmark](https://recommonmark.readthedocs.io/en/latest/index.html)
 처음으로 의심했던 건 >>*markdown parsing 과정에서 non-ascii character의 누락 가능성*<< 이었다.
 하지만 정상적으로 parsing 이후에도 heading text가 유지되는 것을 확인했고, recommonmark는 결백한 친구임을 알아냈다. (parsing 과정에서는 id, internal link가 생성되지 않는다)
 
## [sphinx](https://github.com/sphinx-doc/sphinx)
 다음으로 의심한 친구는 sphinx였다. >>*build 과정에서 non-ascii character의 누락 가능성*<<에 중점을 두고 쫓아가봤다. 다음 [코드](https://github.com/sphinx-doc/sphinx/blob/004b68f281dbad6e7598c8c2f56945bb3bc07107/sphinx/writers/html.py)에서 힌트를 발견할 수 있었다.
 
{% highlight python %}
 def depart_title(self, node):
        # type: (nodes.Node) -> None
        close_tag = self.context[-1]
        if (self.permalink_text and self.builder.add_permalinks and
           node.parent.hasattr('ids') and node.parent['ids']):
            # add permalink anchor
            if close_tag.startswith('</h'):
                self.add_permalink_ref(node.parent, _('Permalink to this headline'))
            elif close_tag.startswith('</a></h'):
                self.body.append(u'</a><a class="headerlink" href="#%s" ' %
                                 node.parent['ids'][0] +
                                 u'title="%s">%s' % (
                                     _('Permalink to this headline'),
                                     self.permalink_text))
            elif isinstance(node.parent, nodes.table):
                self.body.append('</span>')
                self.add_permalink_ref(node.parent, _('Permalink to this table'))
        elif isinstance(node.parent, nodes.table):
            self.body.append('</span>')
{% endhighlight %}
 
  `href="#%s"`에서 `%s`에 들어가는 문자열은 `node.parent['ids']`, 즉 **class node 'ids' attribute 값**이었다!
  
  여기서 사용되는 `class node`는 sphinx가 정의한 것이 아니라, >>*sphinx가 가져다 쓰고 있는 오픈소스인 [docutils](http://docutils.sourceforge.net)의 자료구조*<<였고 'ids' attribute또한 docutils에서 assign하고 있었다. sphinx 또한 결백한 친구였음을 알아냈다.

## [docutils](http://docutils.sourceforge.net)
 이제 'ids' attribute가 어떤 과정으로 assign되는지 뜯어볼 차례다. `class node`가 정의된 [소스 파일](https://sourceforge.net/p/docutils/code/HEAD/tree/trunk/docutils/docutils/nodes.py)에서 다음과 같은 comment를 확인 할 수 있었다.
  
 >    There are two special attributes: 'ids' and 'names'.  Both are<br>
    lists of unique identifiers, and names serve as human interfaces<br>
    to IDs.  Names are case- and whitespace-normalized (see the<br>
    fully_normalize_name() function), and IDs conform to the regular<br>
    expression `[a-z](-?[a-z0-9]+)*` (see the make_id() function).
 
 ***범인을 찾았다!***
 
 위에서 찾았던 [규칙](#rules)이 얼추 맞았다. case- and whitespace-normalized 이후 regex (`[a-z](-?[a-z0-9]+)*`)에 맞는 형식으로 'ids' attribute를 assign하고 있었다.
 
 [make_id()](https://sourceforge.net/p/docutils/code/HEAD/tree/trunk/docutils/docutils/nodes.py#l2093) 함수를 보면 보다 명확하게 확인할 수 있다.
 
 {% highlight python %}
 def make_id(string):
    """
    Convert `string` into an identifier and return it.

    Docutils identifiers will conform to the regular expression
    ``[a-z](-?[a-z0-9]+)*``.  For CSS compatibility, identifiers (the "class"
    and "id" attributes) should have no underscores, colons, or periods.
    Hyphens may be used.

    - The `HTML 4.01 spec`_ defines identifiers based on SGML tokens:

          ID and NAME tokens must begin with a letter ([A-Za-z]) and may be
          followed by any number of letters, digits ([0-9]), hyphens ("-"),
          underscores ("_"), colons (":"), and periods (".").

    - However the `CSS1 spec`_ defines identifiers based on the "name" token,
      a tighter interpretation ("flex" tokenizer notation; "latin1" and
      "escape" 8-bit characters have been replaced with entities)::

          unicode     \\[0-9a-f]{1,4}
          latin1      [&iexcl;-&yuml;]
          escape      {unicode}|\\[ -~&iexcl;-&yuml;]
          nmchar      [-a-z0-9]|{latin1}|{escape}
          name        {nmchar}+

    The CSS1 "nmchar" rule does not include underscores ("_"), colons (":"),
    or periods ("."), therefore "class" and "id" attributes should not contain
    these characters. They should be replaced with hyphens ("-"). Combined
    with HTML's requirements (the first character must be a letter; no
    "unicode", "latin1", or "escape" characters), this results in the
    ``[a-z](-?[a-z0-9]+)*`` pattern.

    .. _HTML 4.01 spec: http://www.w3.org/TR/html401
    .. _CSS1 spec: http://www.w3.org/TR/REC-CSS1
    """
    id = string.lower()
    if not isinstance(id, unicode):
        id = id.decode()
    id = id.translate(_non_id_translate_digraphs)
    id = id.translate(_non_id_translate)
    # get rid of non-ascii characters.
    # 'ascii' lowercase to prevent problems with turkish locale.
    id = unicodedata.normalize('NFKD', id).\
         encode('ascii', 'ignore').decode('ascii')
    # shrink runs of whitespace and replace by hyphen
    id = _non_id_chars.sub('-', ' '.join(id.split()))
    id = _non_id_at_ends.sub('', id)
    return str(id)
 {% endhighlight %}

comment에서 `HTML 4.01 spec`에서 정의한 identifier 형식을 만족시키기 위해 이런 과정을 거쳐야 함을 간략하게 설명해준다.

그럼 `[a-z](-?[a-z0-9]+)*` 패턴을 만족하지 못하는 경우는 쓸쓸히 empty identifier를 부여받고 평생 anonymous로 살아가야 할까? 😢

## Unicode string to byte stream
 ascii character를 1byte(`0x00~0x7F`)로 표현할 수 있는 것처럼, unicode 또한 >>*16진수로 이루어진 byte stream*<<으로 표현할 수 있다. (ex. `페퍼로니` to `0xD3980xD37C0xB85C0xB2C8`) 
 
 ***"그렇다면 unicode byte stream을 string으로 떠서 indentifier로 사용할 수 있지 않을까?"***
 
 python2에서 unicode의 byte stream을 string으로 만드는 건 간단하게 할 수 있다.
 
 {% highlight python %}
 byte_stream_str = "".join(map(lambda x: hex(ord(x)), unicode_string))
 {% endhighlight %}

unicode 문자열 각 글자를 16진수로 나타낸 다음, 전부 이어붙이면 된다. (위 코드)

이렇게 만들어낸 문자열을 바로 identifier로 사용하고 싶지만, `(0x[0-9a-f])+` 꼴이기 때문에 `HTML 4.01 spec`에서 정의한 identifier 형식 `must begin with a letter ([A-Za-z])`를 만족시키기 위해선 문자열의 첫 부분을 알파벳으로 바꿔줘야 한다.

패치 작성시에 `.replace('0x', 'u')`를 추가해 `0x`를 `u`로 바꾸는 방식을 적용했으나 (최종 문자열은 `(u[0-9a-f])+`꼴이 된다), 문자열의 첫 글자를 알파벳으로 바꾸거나, 알파벳을 문자열의 첫 부분에 삽입하는 방식으로 진행해도 괜찮다.

## [Patch](https://sourceforge.net/p/docutils/patches/143/)
make_id() 내부에서 **non-ascii character를 전부 제거한 뒤, 빈 문자열만 남게 되는 경우** 위 코드를 통해 **byte stream을 identifier로 return**하도록 수정하여 패치를 제출했다. ([제출한 패치](https://github.com/KOREAN139/docutils_patch_143/blob/master/make_id.patch))

# **Finishing the journey**
nekoyume 프로젝트 구경에서 샛길로 빠져 docutils에 patch submit을 하기까지의 (...) 여정을 간략하게 정리해봤다. 작은 이슈의 원인을 찾아 거꾸로 올라가며 여러 프로젝트의 소스를 분석하고, 각 과정에서 의심점을 생각하고, 그 의심점을 해결해 나가면서 root cause를 찾아내고, 최종적으로 어떻게 이슈를 해결할지 아이디어를 고안해내는 과정이 굉장히 재밌었다.

제출한 패치가 (approve, decline) 중 어느 선택을 받게 될 지 모르겠지만, 아마 approve 되더라도 docutils가 relase를 자주 하지 않는 오픈소스기 때문에 현재 docutils를 사용하는 다른 오픈소스에 적용되기까지는 오랜 시간이 걸리지 않을까 싶다.
