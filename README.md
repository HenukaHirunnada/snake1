# snake1
game
{snake.pas}
{classic game: avoid your tail and walls, go for apples}
{TODO?:	- if press key perpendicular to move + backwards - eats itself}

program SnakeGame;
uses crt;	{ncurses analogue}
const
	DelayDuration = 100;	{movement delay}
	TRIESNUM = 128;			{umber of tries to find a free space}
	MAXHEIGHT = 40;			{heght of a game field}
	MAXWIDTH = 60;			{width of a game field }
	
type
	segment = ^star;
	Direction = (up, down, left, right, stop);
	star = record			{body segment}
		CurX, CurY: integer;
		next: segment;
		trend: Direction
	end;

	AppleStatus = (eated, untouched);
	fruit = record			{apple}
		CurX, CurY: integer;
		status: AppleStatus;
	end;
	papple = ^fruit;

	borders = record		{game field borders}
		zeroX, endX, zeroY, endY: integer;
	end;

procedure BuildBorder(var box: borders);	{draws game field borders}
	var
		i: integer;
	begin
		TextColor(white);	{white walls}
		
		{top wall}
		i:=1;
		while (i < MAXWIDTH) do begin		
			GotoXY((i+box.zeroX),box.zeroY);
			write('_');
			i:=i+1;
		end;
		box.endX:=i+box.zeroX;
		
		{right wall}
		i:=1;
		while (i < MAXHEIGHT) do begin	
			GotoXY(box.endX,(box.zeroY+i));
			write('|');
			i:=i+1;
		end;
		box.endY:=i+box.zeroY-1;

		{bottom wall}
		i:=1;
		while (box.endX - i) > box.zeroX do begin
			GotoXY(box.endX-i,box.endY);
			write('_');
			i:=i+1;
		end;

		{left wall}
		i:=0;
		while (box.endY - i) > box.zeroY do begin
			GotoXY(box.zeroX,(box.endY-i));
			write('|');
			i:=i+1;
		end;
		GotoXY(1,1);		{return pointer to NW corner}
	end;

procedure ShowStar(var s: segment);				{reflect body segment}
	begin
		GotoXY(s^.CurX, s^.CurY);
		TextColor(Green);	{we are green}
		write('*');
		GotoXY(1,1)
	end;

procedure HideStar(var s: segment);				{conceal body segment}
	begin
		GotoXY(s^.CurX, s^.CurY);
		write(' ');
		GotoXY(1,1)
	end;

procedure Grow(var tail, prev: segment);		{adds segments to body}
	var i:integer = 0;
	begin
		while i < 2 do begin {one apple - two segments}
			new(tail^.next);
			tail:=tail^.next;
			prev:=tail;
			i:=i+1;
		end;
		exit;
	end;

procedure SelfBiteTest(var s: segment);			{selfdestruction check}
	var
		pp: ^segment;				{pointer to pointer}
	begin
		pp := @(s^.next);			{s^ is a head}
		while pp^ <> nil do begin	{go throught list}
			if (s^.CurX = pp^^.CurX) AND (s^.CurY = pp^^.CurY) then begin
				writeln('Well done, autocannibal!');
				halt(1)
			end
			else
				pp:=@(pp^^.next)
		end;
	end;

function TestFree(var x,y: integer; h: segment):boolean; {find free space}
	var
		pp: ^segment;			{pointer to pointer}
	begin
		pp := @(h);				{h - head}

		if pp^ = nil then begin	{end check}
			TestFree := true;
			exit;
		end;
		
		{look if apple coordinates match any part of body}
		if (pp^^.CurX = x) AND (pp^^.CurY = y) then begin
			TestFree := false;
			exit;
		end
		else begin
			pp:=@(pp^^.next);
			TestFree(x,y,pp^);	{recursion}
		end;
	end;

procedure EmergeApple(var a: papple; x,y: integer);		{draw an apple}
	begin
		a^.CurX:= x;
		a^.CurY:= y;
		a^.status:=untouched;
		GotoXY(a^.CurX, a^.CurY);
		TextColor(Red);		{red one}
		write('@');
		GotoXY(1,1);			
	end;

procedure CreateApple(var apple: papple; h: segment; box: borders);
{find place for an apple. it must be free from snake.}
	var
		x,y,dx,dy,i,znak, pdx, tmpx, tmpy: integer;		{aarrrghhh!!}
	begin
		Randomize;										{shake dice}
		x := random(box.endX-box.zeroX-2)+box.zeroX+1;	{range of x}
		y := random(box.endY-box.zeroY-2)+box.zeroY+1;	{range of y}
		dx :=0;		{delta x}
		dy :=0;		{delta y}
		pdx:=0;		{previous dx}
		znak := -1;	{even iterations should be with negative x,y}
		i:=1;		{iterator}

		{works like clockwise spiral (0 - start, nums - steps):
		 678
		 501
		 432
		 x+1, y+1, x-2, y-2, x+3, y-3..-4...}
		while abs(i) < TRIESNUM do begin
			{move on x axis}
			while abs(dx) <= abs(pdx) do begin			{abs(x) -> |x|}
				dx:=dx+i;
				tmpx:=x+dx;	{functions unable to take this expressions}
				tmpy:=y+dy;
{!}					write(''); {IF THERE IS NO STRING - NOTHING WORKS WHY??}
{endless cycle of TestFree, becaue it is unable to call EmergeApple.}
				if TestFree(tmpx,tmpy,h) then begin
					EmergeApple(apple,tmpx,tmpy);		
					exit;
				end;
			end;
			{move on y axis}
			while abs(dy) <= abs(dx) do begin
				dy:=dy+i;
				tmpy:=y+dy;	{functions unable to take this expressions}
				tmpx:=x+dx;
				if TestFree(tmpx,tmpy,h) then begin
					EmergeApple(apple, tmpx, tmpy);
					exit;
				end;
			end;
			{prepare for new circle}
			y:=y+dy;
			x:=x+dx;
			pdx:=dx;
			i:=i*znak;	{positive\negative switch}
			dy:=0;
			dx:=0;
		end;
	end;

function AppleTest(var s: segment; a: papple): boolean;	{check if we ate it}
	begin
		if (s^.CurX = a^.CurX) AND (s^.CurY = a^.CurY) then
			AppleTest:= true
		else
			AppleTest:=false
	end;

procedure MoveBody(var s: segment; prev: segment);		{move snake}
	begin
		if s^.next = nil then begin		{end check}
			exit;
		end;
		HideStar(s);					{erase segment}
		MoveBody(s^.next, s);			{recursion}
		s^.CurX  := prev^.CurX;			{every next gets params of his prev}
		s^.CurY  := prev^.CurY;
		s^.trend := prev^.trend;
		ShowStar(s);					{show segment}
	end;

procedure BorderTest(var head: segment; box: borders);	{wall collision check}
	begin
		{if head coordinates match with walls - game over}
		if (head^.CurX = box.zeroX) OR (head^.CurX = box.endX) then begin
			writeln('Whatch tour step');
			halt(1);
		end
		else
		if (head^.CurY = box.zeroY) OR (head^.CurY = box.endY) then begin
			writeln('Whatch tour step');
			halt(1);
		end;
	end;

procedure MoveHead(var head, tail, prev: segment; apple: papple); {NUFF said}
	{head is special case, because it selects direction}
	begin
		HideStar(head);						{erase head}
		if head^.next <> nil then begin		{if we have some body}
			MoveBody(head^.next, head);		{move it}
		end;
		
		{if head moves in some direction, it advances}
		case head^.trend of
			left:	head^.CurX := head^.CurX-1;
			right:	head^.CurX := head^.CurX+1; 
			up:		head^.CurY := head^.CurY-1; 
			down:	head^.CurY := head^.CurY+1; 
		end;
		ShowStar(head);						{draw head}
	end;

procedure HandleArrowKey(var head: segment; extcode: char);	{snake control}
{move in direction of pressed key}
	begin
		case extcode of
			#75: { left }
				if head^.trend <> right then
					head^.trend:=left;
			#77: { right }
				if head^.trend <> left then
					head^.trend:=right;
			#72: { up }
				if head^.trend <> down then
					head^.trend:=up;
			#80: { down }
				if head^.trend <> up then
					head^.trend:=down;
			' ': { stop moving }
				head^.trend:=stop;
		end
	end;

procedure ShowScore(var score: integer);	{show eated apples}
	begin
		TextColor(white);					{white color}
		GotoXY(2, (ScreenHeight div 2));	{nice place}
		score:= score+1;					{apple iteration}
		write('SCORE: ',score);
		GotoXY(1,1);						{back in corner}
	end;

var
	ch: char;
	head,tail,prev: segment;
	apple: papple;
	box: borders;
	score: integer;

begin
	clrscr;			{clean screen}
	{calculate box coordinates}
	box.zeroY:= ((ScreenHeight div 2) - (MAXHEIGHT div 2));
	box.zeroX:= ((ScreenWidth div 2) - (MAXWIDTH div 2));
	
	{create snake}
	new(head);
	tail:=head;
	prev:=head;
	head^.CurX := ScreenWidth div 2;
	head^.CurY := ScreenHeight div 2;
	head^.trend := stop;	
	head^.next:=nil;
	
	{create apple}
	new(apple);
	apple^.status:=eated;
	apple^.curX:=0;
	apple^.curY:=0;
	
	{you start with -1 score =]}
	score:= -1;

	BuildBorder(box);
	ShowScore(score);
	ShowStar(head);
	CreateApple(apple, head, box);
	
	{main cycle}
	while true do begin
		{if we are not pressing keys}
		if not KeyPressed then begin
			if head^.trend <> stop then
				MoveHead(head, tail, prev, apple);
				BorderTest(head, box);
				SelfBiteTest(head);
				if AppleTest(head, apple) then begin
					Grow(tail, prev);
					ShowScore(score);
					CreateApple(apple, head, box)
				end;
			delay(DelayDuration);
			continue;
		end;
		
		{if we are trying to control snake}
		ch := ReadKey;
		case ch of
			#0: begin
				ch := ReadKey;			{ get extended code }
				HandleArrowKey(head, ch)
			end;
			#27: break; 				{ esc for EXIT }
{$IFDEF DEBUG}
			#13: Grow(tail, prev);		{ for tests} 
			' ': head^.trend := stop;	{ stop moving }
{$ENDIF}
		end
	end;
	clrscr;								{clean screen}
	TextColor(LightGray);				{back to normal color}
end.
