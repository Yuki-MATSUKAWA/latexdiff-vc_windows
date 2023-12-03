# latexdiff-vc_windows
English follows Japanese.

## 日本語での説明 --- Japanese ---
latexdiff-vc を Windows 環境で使用する際に生じるエラーに対処したファイルです．

### ファイル説明
* `latexdiff-vc_texlive_original.pl`: TeX Live 2023に標準で入っている `latexdiff-vc.pl` をそのまま上げています．復旧用に使ってください．
* `latexdiff-vc.pl`: Windows 環境で使用する際にエラーが出ないようデバッグしたファイルです．自分でコードを書き換えたくない人はこのファイルを使用してください．

### 使い方
難しい説明されてもわからないから早く使い方を教えてほしいという方はここと次の節を読んでください．
以降の操作を行う前に，ご自身の Windows PC に TeX 環境が整っていることを確認してください．
確認ができたら，`latexdiff-vc_widows` 丸ごとでもいいですし `latexdiff-vc.pl` 単体でも構わないので，ご自身の Windows PC にダウンロードしてください．
ダウンロードした `latexdiff-vc.pl` は `C:\texlive\<year>\texmf-dist\scripts\latexdiff` 内に入れて，既に入っている `latexdiff-vc.pl` を置き換えてください．
`<year>` には TeX Live のバージョンが入ります．
TeX Live 2023 を使用している場合は `2023` が入ります．
ファイルを置き換えたらもう使える状態になっています．

### 実行時の注意点
```
latexdiff-vc -e utf8 -t CFONT --flatten --git --force -r HEAD^ main.tex
```
などのコマンドを使用して実行すると思います．
ただし，ここで生成されるファイル名は `main-diffHEAD^.tex` となります．
また，同様に `HEAD^` を `HEAD~` や `HEAD~2` で指定したときは `main-diffHEAD~` や `main-diffHEAD~2` のようになります．
ターミナル上で，ファイル名に `^` や `~` が入っているファイルを操作するとエラーの原因になり得るので，それぞれ `-` と `_` で置換するようにしています．
`latexdiff-vc` 実行後のファイル操作の際にはご注意ください．
例えば次のように置換されます．

| 指定するコミット | 出力されるファイル名 |
| ------ | --------------- |
| HEAD^ | main-diffHEAD- |
| HEAD^^ | main-diffHEAD-- |
| HEAD^2 | main-diffHEAD-2 |
| HEAD~ | main-diffHEAD- |
| HEAD~~ | main-diffHEAD-- |
| HEAD~2 | main-diffHEAD-2 |

ハッシュ値で指定する場合は `^` も `~` も含まないためそのまま出力されます．

### デバッグの詳細

#### オプションの渡し方を変更
```perl:diff_latexdiff-vc.pl
  # Remaining options are passed to latexdiff
  if (scalar(@ldoptions) > 0 ) {
-   $options = "\'" . join("\' \'",@ldoptions) . "\'";
+   # Change the way options are passed.
+   $options = join(" ",@ldoptions);
  } else {
    $options = "";
  }
```

#### `checkout_dir` 内の内容を修正
```perl:diff_latexdiff-vc.pl
sub checkout_dir {
  my ($rev,$dirname)=@_;

+ # Convert Windows-style paths to Unix-style.
+ $dirname =~ s/\\/\//g;
+
  unless (-e $dirname) { mkpath([ $dirname ]) or die "Cannot mkdir $dirname ." ;}
  if ( $vc eq "SVN" ) {
    system("svn checkout -r $rev $rootdir $dirname")==0 or die "Something went wrong in executing:  svn checkout -r $rev $rootdir $dirname";
  } elsif ( $vc eq "GIT" ) {
    $rev="HEAD" if length($rev)==0;
-   system("git archive --format=tar $rev | ( cd $dirname ; tar -xf -)")==0 or die "Something went wrong in executing:  git archive --format=tar $rev | ( cd $dirname ; tar -xf -)";
+   # Set the name of the temporary tar file.
+   my $tarfile = "temp_archive.tar";
+   # Create an archive using the git archive command.
+   system("git archive --format=tar $rev > $tarfile")==0 or die "Failed to create archive using: git archive --format=tar $rev > $tarfile";
+   # Extract the archive to a specified directory using the tar command.
+   system("tar -xf $tarfile -C $dirname")==0 or die "Failed to extract $tarfile to $dirname";
+   # Delete the temporary files.
+   unlink $tarfile or warn "Could not remove temporary file $tarfile: $!";
  } elsif ( $vc eq "HG" ) {
    system("hg archive --type files -r $rev $dirname")==0 or die "Something went wrong in executing:  hg archive --type files -r $rev $dirname";
  } else {
```



### 参考文献
* [GitHub: thesis_template_ou_es，『`latexdiff` を用いた差分管理』](https://github.com/ryo-ARAKI/thesis_template_ou_es#latexdiff-%E3%82%92%E7%94%A8%E3%81%84%E3%81%9F%E5%B7%AE%E5%88%86%E7%AE%A1%E7%90%86)
* [Qiita: git+latexdiff-vcで快適にHEADとの差分を確認しながら論文を執筆する for Windows](https://qiita.com/take_me/items/e49c544f9298f936b8fd)



## 英語での説明 --- English ---
This is a file that fixes errors encountered when using latexdiff-vc in a Windows environment.
