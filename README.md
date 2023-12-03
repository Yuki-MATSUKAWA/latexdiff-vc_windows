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
| `HEAD^` | `main-diffHEAD-` |
| `HEAD^^` | `main-diffHEAD--` |
| `HEAD^2` | `main-diffHEAD-2` |
| `HEAD~` | `main-diffHEAD_` |
| `HEAD~~` | `main-diffHEAD__` |
| `HEAD~2` | `main-diffHEAD_2` |

ハッシュ値で指定する場合は `^` も `~` も含まないためそのまま出力されます．

### デバッグの詳細
デバッグの詳細を説明します．
興味ない人は読まなくて大丈夫です．

#### オプションの渡し方を変更
配列の各要素をスペースで結合しました．
オプションがスペースを含まない場合はこの処理で問題ありません．
```diff
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
`checkout_dir` の内容を修正しました．
`tar` は Unix スタイルのパス区切り（`/`）を期待している可能性があるため，Windows スタイルのパス区切り（`\`）を Unix スタイルに変更しました．
また，パイプ（`|`）を使用せずにコマンドを分割することで問題を回避しました．
```diff
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

#### `^` と `~` を `-` と `_` で置換
この変更は行わなくても，`latexdiff-vc` 自体を実行することは可能です．
ただし，出力されるファイル名に `^` や `~` が入っているとその後のファイル操作でエラーが出る可能性があるので，`-` と `_` に置換するようにしました．
```diff
if ( scalar(@revs) == 2 ) {
- $append = "-diff$revs[0]-$revs[1]";
+ # Replace each '^' with '-' and each '~' with '_'
+ my $rev0 = $revs[0];
+ my $rev1 = $revs[1];
+ $rev0 =~ s/\^/-/g;
+ $rev1 =~ s/\^/-/g;
+ $rev0 =~ s/~/_/g;
+ $rev1 =~ s/~/_/g;
+
+ $append = "-diff$rev0-$rev1";
} elsif ( scalar(@revs) == 1 || $revs[0] ) {
- $append = "-diff$revs[0]";
+ # Replace each '^' with '-' and each '~' with '_'
+ my $rev = $revs[0];
+ $rev =~ s/\^/-/g;
+ $rev =~ s/~/_/g;
+
+ $append = "-diff$rev";
} else {
  $append = "-diff";
}
```


## 英語での説明 --- English ---
This is a file that fixes errors encountered when using latexdiff-vc in a Windows environment.

### File Description
* `latexdiff-vc_texlive_original.pl`: This is the original latexdiff-vc.pl that comes standard with TeX Live 2023. Please use it for recovery purposes.
* `latexdiff-vc.pl`: This is a debugged file for use in Windows environments to prevent errors. If you do not want to modify the code yourself, please use this file.

### How to Use
For those who find complicated explanations difficult to understand and want to quickly learn how to use this, please read this section and the next.
Before proceeding with the following steps, please ensure that your Windows PC has a TeX environment set up.
Once confirmed, you can download the entire `latexdiff-vc_windows` package or just the `latexdiff-vc.pl` file to your Windows PC.
Place the downloaded `latexdiff-vc.pl` in `C:\texlive\<year>\texmf-dist\scripts\latexdiff`, replacing the existing `latexdiff-vc.pl` that is already there.
Replace `<year>` with the version of your TeX Live.
For instance, if you are using TeX Live 2023, it will be `2023`.
Once the file is replaced, it is ready to use.

### Points to Note When Executing
You may use commands like this for execution:
```
latexdiff-vc -e utf8 -t CFONT --flatten --git --force -r HEAD^ main.tex
```
However, the generated file name will be `main-diffHEAD^.tex`.
Similarly, if you specify `HEAD^` as `HEAD~` or `HEAD~2`, the file names will be `main-diffHEAD~` or `main-diffHEAD~2`, respectively.
In the terminal, having `^` or `~` in file names can cause errors, so they are replaced with `-` and `_` respectively.
Please be cautious when handling files after running `latexdiff-vc`.
For example, the replacements will be as follows:

| Specified Commit | Output File Name |
| ------ | --------------- |
| `HEAD^` | `main-diffHEAD-` |
| `HEAD^^` | `main-diffHEAD--` |
| `HEAD^2` | `main-diffHEAD-2` |
| `HEAD~` | `main-diffHEAD_` |
| `HEAD~~` | `main-diffHEAD__` |
| `HEAD~2` | `main-diffHEAD_2` |

When specifying with a hash value, as it does not contain `^` or `~`, it will be output as it is.

### Details of the Debugging
I will explain the details of the debugging.
Those who are not interested do not need to read this.

#### Changing How Options Are Passed
I concatenated each element of the array with spaces.
If the options do not contain spaces, there is no problem with this process.
```diff
  # Remaining options are passed to latexdiff
  if (scalar(@ldoptions) > 0 ) {
-   $options = "\'" . join("\' \'",@ldoptions) . "\'";
+   # Change the way options are passed.
+   $options = join(" ",@ldoptions);
  } else {
    $options = "";
  }
```

#### Modifying the Contents within `checkout_dir`
I modified the contents of `checkout_dir`.
Since the tar command may expect Unix-style path separators (`/`), I changed the Windows-style path separators (`\`) to Unix-style.
Also, I avoided the issue by splitting the command without using a pipe (`|`).
```diff
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

#### Replacing `^` and `~` with `-` and `_`
It is possible to run `latexdiff-vc` itself without making this change.
However, since the file names generated by the output include `^` and `~`, which can cause errors in subsequent file operations, I have replaced them with `-` and `_`.
```diff
if ( scalar(@revs) == 2 ) {
- $append = "-diff$revs[0]-$revs[1]";
+ # Replace each '^' with '-' and each '~' with '_'
+ my $rev0 = $revs[0];
+ my $rev1 = $revs[1];
+ $rev0 =~ s/\^/-/g;
+ $rev1 =~ s/\^/-/g;
+ $rev0 =~ s/~/_/g;
+ $rev1 =~ s/~/_/g;
+
+ $append = "-diff$rev0-$rev1";
} elsif ( scalar(@revs) == 1 || $revs[0] ) {
- $append = "-diff$revs[0]";
+ # Replace each '^' with '-' and each '~' with '_'
+ my $rev = $revs[0];
+ $rev =~ s/\^/-/g;
+ $rev =~ s/~/_/g;
+
+ $append = "-diff$rev";
} else {
  $append = "-diff";
}
```

## 参考文献 References (in Japanese)
* [GitHub: thesis_template_ou_es，『`latexdiff` を用いた差分管理』](https://github.com/ryo-ARAKI/thesis_template_ou_es#latexdiff-%E3%82%92%E7%94%A8%E3%81%84%E3%81%9F%E5%B7%AE%E5%88%86%E7%AE%A1%E7%90%86)
* [Qiita: git+latexdiff-vcで快適にHEADとの差分を確認しながら論文を執筆する for Windows](https://qiita.com/take_me/items/e49c544f9298f936b8fd)
